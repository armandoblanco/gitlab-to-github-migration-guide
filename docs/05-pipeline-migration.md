# Fase 5: Migración de Pipelines CI/CD

## Migración automatizada con GitHub Actions Importer

GitHub Actions Importer es la herramienta oficial para convertir pipelines de GitLab a GitHub Actions de forma automatizada. Genera un **Pull Request** con el workflow convertido.

> **Prerequisito:** Haber completado el [Assessment](04-pipeline-assessment.md) con `audit` y `dry-run`.

### Opción A: Migración directa (`migrate`)

El comando `migrate` convierte el pipeline y abre un **Pull Request** en el repositorio de GitHub:

```bash
gh actions-importer migrate gitlab \
  --target-url https://github.com/ORG/REPO \
  --output-dir tmp/migrate \
  --namespace mi-namespace-gitlab \
  --project mi-proyecto
```

**Salida esperada:**

```
[2024-01-15 10:30:20] Logs: 'tmp/migrate/log/actions-importer-20240115-103020.log'
[2024-01-15 10:30:20] Pull request: 'https://github.com/ORG/REPO/pull/1'
```

El Pull Request incluye:
- **Workflow convertido** en `.github/workflows/`
- **Sección "Manual steps"** con pasos que debes completar manualmente (crear secrets, configurar runners, etc.)

### Opción B: Dry-run → revisión → merge manual

Si prefieres revisar antes de crear el PR:

```bash
# 1. Generar el workflow localmente
gh actions-importer dry-run gitlab \
  --output-dir tmp/dry-run \
  --namespace mi-namespace-gitlab \
  --project mi-proyecto

# 2. Revisar el workflow generado
cat tmp/dry-run/*.yml

# 3. Copiar a tu repositorio y hacer push manualmente
cp tmp/dry-run/*.yml .github/workflows/
git add .github/workflows/
git commit -m "Migrate CI/CD pipeline to GitHub Actions"
git push
```

### Usar un archivo local

```bash
gh actions-importer dry-run gitlab \
  --output-dir tmp/dry-run \
  --namespace mi-namespace-gitlab \
  --project mi-proyecto \
  --source-file-path .gitlab-ci.yml
```

### Custom transformers

Si Actions Importer no convierte algo correctamente, puedes crear **custom transformers** para personalizar la conversión:

```bash
gh actions-importer dry-run gitlab \
  --output-dir tmp/dry-run \
  --namespace mi-namespace-gitlab \
  --project mi-proyecto \
  --custom-transformers mi-transformer.rb
```

> Documentación completa: [Extending GitHub Actions Importer with custom transformers](https://docs.github.com/en/actions/migrating-to-github-actions/automated-migrations/extending-github-actions-importer-with-custom-transformers)

### Limitaciones de Actions Importer

| Limitación | Solución |
|---|---|
| Valores de variables masked/protected no se migran | Crear manualmente en GitHub Secrets |
| Artifact reports no soportados | Configurar manualmente |
| Cache entre jobs de diferentes workflows | No soportado, usar alternativa |
| `audit` solo funciona con cuenta de organización | Usar `dry-run` con cuenta individual |

---

## Complemento: Conversión con GitHub Copilot

Para los items que Actions Importer marca como **parcialmente convertidos** o **no soportados**, usa el agente de Copilot para obtener recomendaciones contextuales.

### Uso del agente para ajustes post-Importer

1. Abrir VS Code con GitHub Copilot
2. Invocar: **`@gitlab-to-github`**
3. Pegar el workflow generado por Actions Importer junto con el `.gitlab-ci.yml` original

```
@gitlab-to-github Revisa este workflow generado por Actions Importer y ajusta
los items marcados como "unsupported" o "partially supported". Este es el
.gitlab-ci.yml original y este es el workflow convertido:

[pegar ambos archivos]
```

El agente complementa Actions Importer:
- **Ajusta** items parcialmente convertidos con contexto del pipeline original
- **Sugiere** alternativas para features no soportadas
- **Aplica** mejores prácticas (permisos mínimos, health checks, versiones pinneadas)
- **Genera** reporte de lo que cambió y por qué

---

## Sintaxis soportada por Actions Importer

La siguiente tabla muestra el estado de conversión automática de cada feature de GitLab:

| Feature de GitLab | Conversión en GitHub Actions | Estado |
|---|---|---|
| `after_script` | `jobs.<job_id>.steps` | Soportado |
| `auto_cancel_pending_pipelines` | `concurrency` | Soportado |
| `before_script` | `jobs.<job_id>.steps` | Soportado |
| `build_timeout` / `timeout` | `jobs.<job_id>.timeout-minutes` | Soportado |
| `default` | N/A | Soportado |
| `image` | `jobs.<job_id>.container` | Soportado |
| `job` | `jobs.<job_id>` | Soportado |
| `needs` | `jobs.<job_id>.needs` | Soportado |
| `resource_group` | `jobs.<job_id>.concurrency` | Soportado |
| `schedule` | `on.schedule` | Soportado |
| `script` | `jobs.<job_id>.steps` | Soportado |
| `stages` | `jobs` | Soportado |
| `tags` | `jobs.<job_id>.runs-on` | Soportado |
| `variables` | `env`, `jobs.<job_id>.env` | Soportado |
| `environment` | `jobs.<job_id>.environment` | Parcial |
| `include` | Merge en un solo grafo de jobs | Parcial |
| `only` / `except` | `jobs.<job_id>.if` | Parcial |
| `parallel` | `jobs.<job_id>.strategy` | Parcial |
| `rules` | `jobs.<job_id>.if` | Parcial |
| `services` | `jobs.<job_id>.services` | Parcial |
| `workflow` | `if` | Parcial |

> Referencia completa: [github/gh-actions-importer — GitLab](https://github.com/github/gh-actions-importer/blob/main/docs/gitlab/index.md)

---

## Tabla de equivalencias (referencia manual)

### Estructura general

| GitLab CI/CD | GitHub Actions |
|---|---|
| `.gitlab-ci.yml` | `.github/workflows/<nombre>.yml` |
| `stages:` + orden implícito | `jobs:` con `needs:` explícitos |
| `job_name:` | `jobs.<job_id>:` |
| `script:` | `steps:` con `run:` |
| `image:` | `container:` o `runs-on:` + `actions/setup-*` |
| `services:` | `services:` (requiere `ports:` y health checks) |
| `before_script:` | Step inicial con `run:` |
| `after_script:` | Step final con `if: always()` |
| `variables:` | `env:` (workflow, job o step) |
| `rules:` / `only:` / `except:` | `on:` triggers + `if:` en jobs/steps |
| `needs:` | `needs:` |
| `dependencies:` | `needs:` + `actions/download-artifact@v4` |
| `artifacts:paths:` | `actions/upload-artifact@v4` |
| `cache:` | `actions/cache@v4` o cache built-in |
| `extends:` | Expandir inline (no hay equivalente) |
| `include:` | Reusable workflows o expandir inline |
| `trigger:` | `workflow_dispatch:` o `repository_dispatch` |
| `when: manual` | `workflow_dispatch:` o environment con protection rules |
| `when: on_failure` | `if: failure()` |
| `when: always` | `if: always()` |
| `when: delayed` | **Sin equivalente** |
| `allow_failure: true` | `continue-on-error: true` |
| `timeout:` | `timeout-minutes:` |
| `retry:` | `nick-fields/retry@v3` o lógica custom |
| `environment:` | `environment:` |
| `tags:` | `runs-on:` con labels |
| `parallel: matrix:` | `strategy: matrix:` |
| `coverage:` | **Sin equivalente nativo** |
| `release:` | `softprops/action-gh-release@v2` |
| `pages:` | `actions/deploy-pages@v4` |
| `resource_group:` | `concurrency:` con `cancel-in-progress` |

### Variables predefinidas

| GitLab | GitHub Actions |
|---|---|
| `$CI_COMMIT_REF_NAME` | `${{ github.ref_name }}` |
| `$CI_COMMIT_BRANCH` | `${{ github.ref_name }}` (en push) |
| `$CI_COMMIT_TAG` | `${{ github.ref_name }}` (en tag push) |
| `$CI_COMMIT_SHA` | `${{ github.sha }}` |
| `$CI_COMMIT_SHORT_SHA` | `${GITHUB_SHA::8}` (en script) |
| `$CI_PIPELINE_ID` | `${{ github.run_id }}` |
| `$CI_PIPELINE_IID` | `${{ github.run_number }}` |
| `$CI_PROJECT_NAME` | `${{ github.event.repository.name }}` |
| `$CI_PROJECT_PATH` | `${{ github.repository }}` |
| `$CI_PROJECT_DIR` | `${{ github.workspace }}` |
| `$CI_PROJECT_URL` | `${{ github.event.repository.html_url }}` |
| `$CI_MERGE_REQUEST_IID` | `${{ github.event.pull_request.number }}` |
| `$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` | `${{ github.head_ref }}` |
| `$CI_MERGE_REQUEST_TARGET_BRANCH_NAME` | `${{ github.base_ref }}` |
| `$CI_REGISTRY_USER` | `${{ github.actor }}` |
| `$CI_REGISTRY_PASSWORD` | `${{ secrets.GITHUB_TOKEN }}` |
| `$CI_REGISTRY_IMAGE` | `ghcr.io/${{ github.repository }}` |
| `$CI_DEFAULT_BRANCH` | `${{ github.event.repository.default_branch }}` |
| `$CI_JOB_TOKEN` | `${{ secrets.GITHUB_TOKEN }}` |
| `$GITLAB_USER_LOGIN` | `${{ github.actor }}` |
| `$GITLAB_USER_EMAIL` | `${{ github.event.pusher.email }}` |

### Templates de GitLab → Actions equivalentes

| Template de GitLab | GitHub Actions |
|---|---|
| `Security/SAST.gitlab-ci.yml` | `github/codeql-action/analyze@v3` |
| `Security/Dependency-Scanning.gitlab-ci.yml` | `actions/dependency-review-action@v4` |
| `Security/Secret-Detection.gitlab-ci.yml` | `trufflesecurity/trufflehog@v3` |
| `Security/Container-Scanning.gitlab-ci.yml` | `aquasecurity/trivy-action@master` |
| `Deploy/ECS.gitlab-ci.yml` | `aws-actions/amazon-ecs-deploy-task-definition@v2` |
| `Deploy/Cloud-Run.gitlab-ci.yml` | `google-github-actions/deploy-cloudrun@v2` |

---

## Diferencias clave

### Modelo de ejecución

- **GitLab**: Stages secuenciales; jobs dentro del mismo stage en paralelo
- **GitHub Actions**: Jobs en paralelo por defecto; usar `needs:` para orden

### Servicios

- **GitLab**: Acceso por nombre de host del contenedor (ej: `postgres`)
- **GitHub Actions**: Acceso por `localhost` con puertos mapeados

### Artifacts

- **GitLab**: Se pasan automáticamente entre stages con `dependencies:`
- **GitHub Actions**: Se requiere `upload-artifact` y `download-artifact` explícitamente

### Variables sensibles

- **GitLab**: `Settings > CI/CD > Variables` (masked/protected)
- **GitHub Actions**: **Secrets** (`${{ secrets.NOMBRE }}`) y **Variables** (`${{ vars.NOMBRE }}`)

---

## Mejores prácticas al convertir

1. **Versiones pinneadas**: `actions/checkout@v4`, nunca `@main`
2. **Permisos explícitos**: Agregar `permissions:` con mínimo privilegio
3. **Health checks en servicios**: Siempre incluir `options:` con health checks
4. **Cache integrado**: Preferir el cache de `actions/setup-node`, `setup-python`, etc.
5. **Docker**: Usar `docker/setup-buildx-action@v3` + `docker/build-push-action@v5` con cache GHA

---

## Ejemplo: Pipeline completo

### GitLab CI/CD

```yaml
stages:
  - build
  - test
  - deploy

variables:
  NODE_VERSION: "20"

build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/

test:
  stage: test
  image: node:${NODE_VERSION}
  needs: [build]
  services:
    - postgres:15
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
  script:
    - npm ci
    - npm test

deploy:
  stage: deploy
  needs: [test]
  script:
    - echo "Deploying..."
  environment:
    name: production
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

### GitHub Actions equivalente

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

env:
  NODE_VERSION: "20"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 1

  test:
    runs-on: ubuntu-latest
    needs: build
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"
      - run: npm ci
      - run: npm test
        env:
          DATABASE_URL: postgres://testuser:testpass@localhost:5432/testdb
```

### Deploy manual (workflow separado)

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: "Escribe 'deploy' para confirmar"
        required: true
        type: string

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.event.inputs.confirm == 'deploy'
    environment:
      name: production
    steps:
      - uses: actions/checkout@v4
      - run: echo "Deploying..."
```

---

## Ejemplo: Docker Build & Push

### GitLab

```yaml
docker-build:
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

### GitHub Actions

```yaml
name: Docker Build and Push

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Dónde colocar los workflows

```
mi-repo/
└── .github/
    └── workflows/
        ├── ci.yml                    # Pipeline principal
        ├── deploy-production.yml     # Deploy manual
        └── docker.yml                # Build de contenedores
```

---

## Siguiente paso

[Fase 6: Post-Migración](06-post-migration.md)
