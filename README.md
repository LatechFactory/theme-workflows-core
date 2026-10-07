# theme-workflows-core

Workflows **reutilizables** (`workflow_call`) del sistema de themes de Latech. La lógica
vive acá, **una sola vez**; cada repo de store la invoca con *thin callers* mínimos. Así
se mantiene en un solo lugar y sirve para todos los stores.

## Workflows

| Reusable | Qué hace | Lo dispara (en el store) |
|---|---|---|
| `new-theme.yml` | Crea un theme nuevo (nace del live), branch + registro | `workflow_dispatch` (y el tool `create_theme` de eve) |
| `push-on-commit.yml` | Deploya el diff de una branch `theme/*` al theme (nunca al live) | `push` a `theme/**` |
| `mirror.yml` | Pulea el live del store a `main` (el espejo) | `schedule` + `workflow_dispatch` |
| `adopt-theme.yml` | Adopta un theme creado en el admin (branch + registro, `origin: manual`) | `repository_dispatch` (webhook) + `workflow_dispatch` |
| `merge.yml` | Dry run de merge entre dos themes (etapa 4a): reporta conflicto y archivos, no pushea | `workflow_dispatch` |
| `merge-apply.yml` | Merge real (etapa 4b): push scopeado al destino o link de compare ante conflicto | `workflow_dispatch` |

## Contrato (cómo invocarlas desde un store)

Cada reusable corre con `actions/checkout` **sin `repository:`**, así que opera sobre el
repo del **caller** = el store (sus branches, su `themes.json`). El caller pasa:

- **`inputs.shop_domain`** — el dominio del store (`clientX.myshopify.com`), desde
  `vars.SHOP_DOMAIN` del repo del store.
- **`secrets: inherit`** — para que la reusable use el `SHOPIFY_THEME_TOKEN` del store.

### Ejemplo de thin caller (en el repo del store)

```yaml
# .github/workflows/push-on-commit.yml
name: push-on-commit
on:
  push:
    branches: ["theme/**"]
permissions:
  contents: write
jobs:
  deploy:
    uses: LatechFactory/theme-workflows-core/.github/workflows/push-on-commit.yml@v1
    with:
      branch: ${{ github.ref_name }}
      shop_domain: ${{ vars.SHOP_DOMAIN }}
    secrets: inherit
```

```yaml
# .github/workflows/new-theme.yml
name: new-theme
on:
  workflow_dispatch:
    inputs:
      name:   { description: "Nombre corto del fix", required: true }
      ticket: { description: "Ticket (opcional)", required: false }
permissions:
  contents: write
jobs:
  create:
    uses: LatechFactory/theme-workflows-core/.github/workflows/new-theme.yml@v1
    with:
      name: ${{ inputs.name }}
      ticket: ${{ inputs.ticket }}
      shop_domain: ${{ vars.SHOP_DOMAIN }}
    secrets: inherit
```

```yaml
# .github/workflows/adopt-theme.yml
name: adopt-theme
on:
  repository_dispatch:
    types: [theme-created]
  workflow_dispatch:
    inputs:
      theme_id: { description: "Theme ID a adoptar", required: true }
permissions:
  contents: write
jobs:
  adopt:
    uses: LatechFactory/theme-workflows-core/.github/workflows/adopt-theme.yml@v1
    with:
      theme_id: ${{ github.event.client_payload.id || inputs.theme_id }}
      shop_domain: ${{ vars.SHOP_DOMAIN }}
    secrets: inherit
```

```yaml
# .github/workflows/mirror.yml
name: mirror-live-theme
on:
  schedule: [{ cron: "0 6 * * *" }]
  workflow_dispatch:
permissions:
  contents: write
jobs:
  pull:
    uses: LatechFactory/theme-workflows-core/.github/workflows/mirror.yml@v1
    with:
      shop_domain: ${{ vars.SHOP_DOMAIN }}
    secrets: inherit
```

## Cada store necesita

- **Secret** `SHOPIFY_THEME_TOKEN` (Theme Access `shptka_` del store).
- **Variable** `SHOP_DOMAIN` (`clientX.myshopify.com`).
- `themes.json` en `main` (arranca en `{}`).

## Versionado

Los thin callers referencian el tag mayor **móvil** `@v1` (mismo patrón que
`actions/checkout@v4`). Cambios compatibles — agregar un workflow, fixes — **mueven**
`v1` al commit nuevo y todos los stores los toman sin tocar sus callers:

```bash
git tag -f v1 && git push -f origin v1
```

Un cambio **incompatible** (romper inputs/outputs) se publica como `v2`; los stores
migran su `@v1` → `@v2` cuando convenga. Así un cambio en core nunca rompe a un store
hasta que mueva su `@vN`.
