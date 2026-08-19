# ci-workflows

Workflows reutilizables de GitHub Actions para los proyectos Ticzar / Siltium (Node).

Hoy vive en local. Cuando lo subas a GitHub (ej. `siltium/ci-workflows`), cada app
lo llama con `uses: OWNER/REPO/.github/workflows/<archivo>.yml@v1`.

## Workflows

| Archivo | Para | Qué hace |
|---------|------|----------|
| `node-security.yml` | Cualquier Node + Dockerfile | Gitleaks + build imagen + Trivy (CRITICAL/HIGH fixables) |
| `next-quality.yml` | Next.js | `npm ci` + build (lint/typecheck opcionales) |
| `nest-api-tests.yml` | Nest / APIs | Newman contra collection Postman |

## Versionado

1. Creá el repo y pusheá estos archivos.
2. Tag inicial: `git tag v1 && git push origin v1` (o `v1.0.0`).
3. En los consumidores usá `@v1` (o el SHA/tag concreto). Evitá `@main` en prod.

En la org de GitHub: Settings → Actions → General → **Access** → permitir que
repos de la org usen workflows de este repo.

## Cómo consumir

En cada app solo queda un orquestador chico. Ver `examples/`.

### API Nest (tickets-api / pay-api)

```yaml
# .github/workflows/ci-qa.yml
name: CI QA
on:
  push:
    branches: [qa]
  workflow_dispatch:

jobs:
  security:
    name: "Stage 1 · Security"
    uses: siltium/ci-workflows/.github/workflows/node-security.yml@v1
    with:
      image-name: ticzar-pay-api

  api-tests:
    name: "Stage 2 · API tests"
    needs: [security]
    uses: siltium/ci-workflows/.github/workflows/nest-api-tests.yml@v1
    with:
      collection-path: postman/collections/TICZAR_Pay_API.postman_collection.json
      environment-path: postman/environments/TICZAR_PAY.postman_environment.json
      working-dir: postman
      folders: |
        login,refresh,register,contacts,webhooks,movement-types,health,countries-read,provinces-read
```

Tickets-api puede omitir `api-tests` si no tiene Newman.

### Next.js (web / admins)

```yaml
# .github/workflows/ci-qa.yml
name: CI QA
on:
  push:
    branches: [qa]
  workflow_dispatch:

jobs:
  security:
    name: "Stage 1 · Security"
    uses: siltium/ci-workflows/.github/workflows/node-security.yml@v1
    with:
      image-name: ticzar-pay-admin

  quality:
    name: "Stage 2 · Quality"
    uses: siltium/ci-workflows/.github/workflows/next-quality.yml@v1
    with:
      env-file: .github/ci.env
      # run-lint: true
      # run-typecheck: true
```

Creá `.github/ci.env` en cada frontend con dummies (Coolify pone las reales en deploy):

```env
NEXT_PUBLIC_API_URL=https://example.invalid
NEXTAUTH_URL=https://example.invalid
NEXTAUTH_SECRET=change-me-ci-only
```

## Qué queda en cada repo de app

- `.github/workflows/ci-qa.yml` — orquestador (10–20 líneas)
- `.github/ci.env` — solo Next, dummies de build
- Se pueden borrar `security.yml` / `quality.yml` / `api-tests.yml` locales
  cuando ya apunten al repo central.

## Inputs (resumen)

### `node-security.yml`
- `image-name` (required)
- `dockerfile` (default `./Dockerfile`)
- `context` (default `.`)
- `gitleaks-image` (default `zricethezav/gitleaks:v8.24.3`)

### `next-quality.yml`
- `node-version` (default `22`)
- `install-command` / `build-command`
- `env-file` (opcional)
- `run-lint` / `run-typecheck` (default `false`)

### `nest-api-tests.yml`
- `collection-path` / `environment-path` (required)
- `working-dir` (default `postman`)
- `folders` (coma o newlines; vacío = collection completa)
- `node-version` (default `20`)
