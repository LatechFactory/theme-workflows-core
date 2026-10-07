# theme-workflows-core

Workflows **reutilizables** (`workflow_call`) del developer agent de themes de Latech. La lógica
vive acá, **una sola vez**; cada repo de store la invoca con *thin callers* mínimos. Así
se mantiene en un solo lugar y sirve para todos los stores.

## Workflows

| Reusable | Qué hace | Lo dispara (en el store) |
|---|---|---|
| `new-theme.yml` | Crea un theme nuevo (nace del live), branch + registro | `workflow_dispatch` (y el tool `create_theme` de eve) |
| `push-on-commit.yml` | Deploya el diff de una branch `theme/*` al theme (nunca al live) | `push` a `theme/**` |
| `mirror.yml` | Pulea el live del store a `main` (el espejo) | `schedule` + `workflow_dispatch` |
| `adopt-theme.yml` | Adopta un theme creado en el admin (branch + registro, `origin: manual`) | `repository_dispatch` (webhook) + `workflow_dispatch` |

> `merge.yml` / `merge-apply.yml` (flujo merchant/dev) se portan en un slice posterior.

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

Las reusables se referencian por tag inmutable: `...@v1`. Al iterar la lógica se publica
un tag nuevo (`v2`, …) y los stores migran cuando conviene — un cambio en core **no**
rompe a los stores hasta que muevan el `@vN`.
