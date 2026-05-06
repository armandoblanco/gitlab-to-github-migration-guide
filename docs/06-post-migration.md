# Fase 6: Post-Migración

## Checklist de validación

Después de migrar el código y los pipelines, verificar:

### Repositorio

- [ ] Todas las ramas están presentes (`git branch -r`)
- [ ] Todos los tags están presentes (`git tag -l`)
- [ ] El historial de commits está completo (`git log --oneline | wc -l`)
- [ ] Los archivos LFS se descargaron correctamente
- [ ] El README se renderiza correctamente
- [ ] Las GitHub Pages funcionan (si aplica)

### Pipelines

- [ ] Los workflows ejecutan correctamente
- [ ] Los services (DB, Redis, etc.) levantan con health checks
- [ ] Los artifacts se generan y descargan correctamente
- [ ] Los secrets están configurados en `Settings > Secrets and variables`
- [ ] Los environments están configurados con protection rules

---

## Configuración de seguridad

### Branch protection rules

```
Settings > Branches > Add branch protection rule
```

| Configuración | Recomendado |
|---|---|
| Branch name pattern | `main` |
| Require a pull request before merging | ✅ |
| Require approvals | ✅ (mínimo 1) |
| Dismiss stale pull request approvals | ✅ |
| Require status checks to pass | ✅ |
| Require branches to be up to date | ✅ |
| Include administrators | ⚠️ Según política |
| Restrict who can push | ⚠️ Según política |

### CODEOWNERS

```bash
# .github/CODEOWNERS
# Propietarios por defecto
* @org/team-leads

# Propietarios por directorio
/src/api/ @org/backend-team
/src/frontend/ @org/frontend-team
/infra/ @org/devops-team
/.github/ @org/devops-team
```

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      dependencies:
        patterns: ["*"]
    assignees:
      - "team-lead"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### GitHub Advanced Security (GHAS)

Si la organización tiene GHAS habilitado:

```
Settings > Code security and analysis
```

- **Dependency graph**: ✅ Habilitar
- **Dependabot alerts**: ✅ Habilitar
- **Dependabot security updates**: ✅ Habilitar
- **Code scanning**: ✅ Configurar con CodeQL
- **Secret scanning**: ✅ Habilitar
- **Push protection**: ✅ Habilitar

---

## Configuración de environments

### Crear environments para deploy

```
Settings > Environments > New environment
```

#### Production

| Configuración | Valor |
|---|---|
| Protection rules | ✅ Required reviewers |
| Reviewers | Lista de aprobadores |
| Wait timer | Opcional (ej: 15 min) |
| Deployment branches | `main` solamente |

#### Staging

| Configuración | Valor |
|---|---|
| Protection rules | Opcional |
| Deployment branches | `main`, `develop` |

### Secrets por environment

```
Settings > Environments > [nombre] > Environment secrets
```

Ejemplo:
- `production` → `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `DB_CONNECTION_STRING`
- `staging` → Mismos keys, diferentes valores

---

## Migración de equipos y permisos

### Roles equivalentes

| GitLab | GitHub |
|---|---|
| Guest | Read |
| Reporter | Triage |
| Developer | Write |
| Maintainer | Maintain |
| Owner | Admin |

### Equipos con IdP (EMU)

Si la organización usa **Enterprise Managed Users**:

1. Los equipos se gestionan desde el IdP (Entra ID, Okta)
2. Crear grupos en el IdP que mapeen a equipos en GitHub
3. Usar SCIM para sincronización automática
4. Los permisos se asignan al equipo, no a usuarios individuales

### Equipos manuales

```
Organization > Teams > New team
```

Asignar repositorios al equipo con el rol apropiado.

---

## Badges y status

Actualizar badges en el README:

```markdown
<!-- Antes (GitLab) -->
[![pipeline](https://gitlab.com/org/repo/badges/main/pipeline.svg)](...)
[![coverage](https://gitlab.com/org/repo/badges/main/coverage.svg)](...)

<!-- Después (GitHub) -->
[![CI](https://github.com/ORG/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/ORG/REPO/actions/workflows/ci.yml)
```

---

## Migración gradual (recomendada)

Para repositorios críticos, se recomienda una migración gradual:

### Fases

```
Fase 1: Migrar código (mirror read-only en GitHub)
  ↓
Fase 2: Configurar pipelines en GitHub (validar en paralelo)
  ↓
Fase 3: Configurar protecciones y permisos
  ↓
Fase 4: Migrar secrets y variables
  ↓
Fase 5: Redirigir nuevos PRs a GitHub
  ↓
Fase 6: Sincronización temporal (dual-write)
  ↓
Fase 7: Corte final (desactivar GitLab)
  ↓
Fase 8: Archivar repositorio en GitLab
```

### Sincronización temporal

Durante la transición, mantener ambos repositorios sincronizados:

```yaml
# .github/workflows/sync-from-gitlab.yml
name: Sync from GitLab

on:
  schedule:
    - cron: "0 */6 * * *"  # Cada 6 horas
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - run: |
          git remote add gitlab https://gitlab.com/org/repo.git || true
          git fetch gitlab
          git merge gitlab/main --no-edit
          git push origin main
```

> **Nota**: Una vez confirmado que GitHub es el origen de verdad, desactivar la sincronización y archivar el repo en GitLab.

---

## Migración de Issues y Merge Requests

### Opciones

| Herramienta | Issues | MRs/PRs | Comentarios | Labels |
|---|---|---|---|---|
| **GitHub Importer** (UI) | ✅ | ❌ | Parcial | ✅ |
| **gl2gh** (CLI) | ✅ | ✅ | ✅ | ✅ |
| **node-gitlab-2-github** | ✅ | ✅ | ✅ | ✅ |
| **Manual** | ✅ | ❌ | ❌ | ✅ |

### Con GitHub CLI

```bash
# Exportar issues de GitLab
# (usar la API de GitLab para extraer y luego crear en GitHub)
gh issue create --title "Migrado: Issue #123" --body "Contenido..."
```

### Recomendación

Para la mayoría de las migraciones, es más práctico:
1. Cerrar issues pendientes en GitLab o migrarlos manualmente
2. Comenzar limpio en GitHub
3. Mantener GitLab como referencia histórica (archivado, read-only)

---

## Runners y ejecución

### GitHub-hosted runners

| Runner | vCPUs | RAM | Almacenamiento |
|---|---|---|---|
| `ubuntu-latest` | 4 | 16 GB | 14 GB SSD |
| `windows-latest` | 4 | 16 GB | 14 GB SSD |
| `macos-latest` | 3 (M1) | 7 GB | 14 GB SSD |

### Self-hosted runners

Si los pipelines de GitLab usaban runners dedicados:

```bash
# Instalar runner self-hosted
# Settings > Actions > Runners > New self-hosted runner

# Linux
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.XXX.X.tar.gz -L https://github.com/actions/runner/releases/download/v2.XXX.X/actions-runner-linux-x64-2.XXX.X.tar.gz
tar xzf ./actions-runner-linux-x64-2.XXX.X.tar.gz
./config.sh --url https://github.com/ORG --token TOKEN
./svc.sh install
./svc.sh start
```

Referencia en el workflow:

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64]
```

---

## IP Allow List (GitHub Enterprise)

Si la organización usa IP Allow List:

```
Organization > Settings > Security > IP allow list
```

Agregar los rangos de IP de:
- Self-hosted runners
- Servicios de deploy (CI/CD externo)
- Oficinas / VPN

---

## Siguiente paso

➡️ [Fase 7: Troubleshooting](07-troubleshooting.md)
