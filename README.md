# Migracion de GitLab a GitHub

## Introduccion

Esta guia documenta el proceso completo para migrar **repositorios de codigo** y **pipelines CI/CD** desde GitLab hacia GitHub. Esta disenada para equipos de DevOps, desarrolladores y lideres tecnicos que necesitan ejecutar migraciones de forma segura, repetible y con minimo riesgo.

### Que cubre esta guia?

- **Migracion de codigo fuente**: ramas, tags, historial de commits y archivos LFS
- **Escaneo y limpieza de secretos**: deteccion y remediacion antes de hacer push a GitHub
- **Assessment de pipelines**: auditoria automatizada con [GitHub Actions Importer](https://docs.github.com/en/actions/tutorials/migrate-to-github-actions/automated-migrations/gitlab-migration) y analisis asistido con GitHub Copilot
- **Conversion de pipelines**: de `.gitlab-ci.yml` a GitHub Actions workflows
- **Post-migracion**: seguridad, permisos, environments y cutover gradual

### Herramientas utilizadas

| Herramienta | Rol en la migracion |
|---|---|
| **GitHub Actions Importer** | Auditoria, forecast y conversion automatizada de pipelines |
| **GitHub Copilot** (`@gitlab-to-github`) | Asistente IA para assessment detallado y ajustes de conversion |
| **gitleaks + BFG** | Escaneo y limpieza de secretos en la historia de commits |
| **GitHub CLI** | Autenticacion, gestion de repositorios y operaciones de migracion |

---

## Parte 1 — Migracion de Repositorios

Clonar el codigo desde GitLab, escanear secretos y hacer push a GitHub.

```mermaid
flowchart TD
    A["Fase 1: Prerrequisitos\nHerramientas, accesos y autenticacion"]
    B["Fase 2: Clonar Repositorio\nClonar desde GitLab y preparar remotes"]
    C["Fase 3: Escaneo de Secretos\nDetectar, limpiar y push a GitHub"]

    A --> B --> C

    style A fill:#2d333b,stroke:#58a6ff,color:#e6edf3
    style B fill:#2d333b,stroke:#58a6ff,color:#e6edf3
    style C fill:#2d333b,stroke:#d29922,color:#e6edf3
```

| # | Fase | Documento | Descripcion |
|---|---|---|---|
| 1 | Prerrequisitos | [01-prerequisites.md](docs/01-prerequisites.md) | Herramientas, accesos, autenticacion y consideraciones para entornos Enterprise (EMU, SAML SSO) |
| 2 | Clonar Repositorio | [02-repository-migration.md](docs/02-repository-migration.md) | Clonar desde GitLab, preparar remotes y metodos de migracion |
| 3 | Escaneo y Push | [03-secret-scanning.md](docs/03-secret-scanning.md) | Escanear secretos, limpiar con BFG si es necesario, y push a GitHub |

### Quick Start — Migrar un repositorio

```bash
# 1. Clonar desde GitLab
git clone git@gitlab.com:GRUPO/REPO.git
cd REPO

# 2. Agregar remote de GitHub (NO hacer push todavia)
git remote add github git@github.com:ORG/REPO.git

# 3. Escanear secretos ANTES de hacer push
gitleaks detect --source . --verbose
# Si hay secretos -> limpiar con BFG (ver docs/03-secret-scanning.md)

# 4. Push a GitHub (solo despues de verificar que no hay secretos)
git push github --all
git push github --tags
```

---

## Parte 2 — Migracion de Pipelines

Evaluar, convertir y validar los pipelines CI/CD de GitLab a GitHub Actions.

```mermaid
flowchart TD
    D["Fase 4: Assessment\nAuditoria con Actions Importer + Copilot"]
    E["Fase 5: Migracion de Pipelines\nConversion a GitHub Actions"]
    F["Fase 6: Post-Migracion\nValidacion, seguridad y cutover"]
    G["Fase 7: Troubleshooting\nErrores comunes y soluciones"]

    D --> E --> F
    F -.-> G

    style D fill:#2d333b,stroke:#a371f7,color:#e6edf3
    style E fill:#2d333b,stroke:#a371f7,color:#e6edf3
    style F fill:#2d333b,stroke:#3fb950,color:#e6edf3
    style G fill:#2d333b,stroke:#f85149,color:#e6edf3
```

| # | Fase | Documento | Descripcion |
|---|---|---|---|
| 4 | Assessment | [04-pipeline-assessment.md](docs/04-pipeline-assessment.md) | Auditoria automatizada con Actions Importer (`audit`, `forecast`) y analisis con Copilot |
| 5 | Migracion de Pipelines | [05-pipeline-migration.md](docs/05-pipeline-migration.md) | Conversion con Actions Importer (`migrate`), ajustes con Copilot, tablas de equivalencias y ejemplos |
| 6 | Post-Migracion | [06-post-migration.md](docs/06-post-migration.md) | Checklist de validacion, seguridad, proteccion de ramas, environments y permisos |
| 7 | Troubleshooting | [07-troubleshooting.md](docs/07-troubleshooting.md) | Errores comunes: SAML SSO, push protection, refs rechazados, atribucion de commits |

### Flujo de herramientas

```mermaid
flowchart LR
    subgraph Assessment
        A1["gh actions-importer audit\nAuditoria masiva"]
        A2["gh actions-importer forecast\nPronostico de runners"]
        A3["@gitlab-to-github\nAnalisis con Copilot"]
    end

    subgraph Conversion
        B1["gh actions-importer dry-run\nPreview local"]
        B2["gh actions-importer migrate\nPR automatico"]
        B3["@gitlab-to-github\nAjustes manuales"]
    end

    A1 --> A2 --> B1
    A3 -.-> B3
    B1 --> B2
    B2 -.-> B3

    style A1 fill:#1f2937,stroke:#58a6ff,color:#e6edf3
    style A2 fill:#1f2937,stroke:#58a6ff,color:#e6edf3
    style A3 fill:#1f2937,stroke:#a371f7,color:#e6edf3
    style B1 fill:#1f2937,stroke:#3fb950,color:#e6edf3
    style B2 fill:#1f2937,stroke:#3fb950,color:#e6edf3
    style B3 fill:#1f2937,stroke:#a371f7,color:#e6edf3
```

### Quick Start — Migrar un pipeline

```bash
# 1. Instalar y configurar Actions Importer
gh extension install github/gh-actions-importer
gh actions-importer configure

# 2. Auditar el namespace completo
gh actions-importer audit gitlab --output-dir tmp/audit --namespace MI-NAMESPACE

# 3. Dry-run de un proyecto especifico
gh actions-importer dry-run gitlab --output-dir tmp/dry-run --namespace MI-NAMESPACE --project MI-PROYECTO

# 4. Migrar (crea un PR con el workflow convertido)
gh actions-importer migrate gitlab --target-url https://github.com/ORG/REPO --output-dir tmp/migrate --namespace MI-NAMESPACE --project MI-PROYECTO
```

### Complementar con GitHub Copilot

1. Abrir el archivo `.gitlab-ci.yml` en VS Code
2. Invocar el agente de Copilot: **`@gitlab-to-github`**
3. Pegar o referenciar el pipeline y obtener ajustes para items parcialmente convertidos

> Ver [Assessment de Pipelines](docs/04-pipeline-assessment.md) para el proceso completo.

---

## Agente de GitHub Copilot

Este repositorio incluye un **agente de GitHub Copilot** (`@gitlab-to-github`) especializado en migracion de pipelines GitLab CI/CD a GitHub Actions. Funciona como **complemento** de GitHub Actions Importer para los ajustes que requieren contexto humano.

```mermaid
flowchart LR
    U["Developer"] -->|"@gitlab-to-github"| C["Copilot Agent"]
    C -->|"Analiza"| GL[".gitlab-ci.yml"]
    C -->|"Genera"| GH[".github/workflows/*.yml"]
    C -->|"Reporta"| R["Advertencias\nLimitaciones\nAcciones manuales"]

    style U fill:#2d333b,stroke:#58a6ff,color:#e6edf3
    style C fill:#2d333b,stroke:#a371f7,color:#e6edf3
    style GL fill:#2d333b,stroke:#d29922,color:#e6edf3
    style GH fill:#2d333b,stroke:#3fb950,color:#e6edf3
    style R fill:#2d333b,stroke:#f85149,color:#e6edf3
```

### Ubicacion

```
.github/agents/gitlab-to-github.agents.md
```

### Uso en VS Code

1. Abre VS Code con GitHub Copilot habilitado
2. En el chat de Copilot, invoca el agente: **`@gitlab-to-github`**
3. Adjunta o pega tu archivo `.gitlab-ci.yml`
4. El agente genera:
   - Workflow(s) de GitHub Actions equivalente(s)
   - Reporte de migracion con advertencias y limitaciones
   - Instrucciones de implementacion

### Capacidades del agente

- Transformacion de stages, jobs, variables, rules, services, artifacts y cache
- Mapeo de variables predefinidas de GitLab (`$CI_*`) a contextos de GitHub
- Identificacion de features sin equivalente directo
- Aplicacion automatica de mejores practicas (versiones pinneadas, permisos minimos, health checks)
- Reporte detallado de problemas, advertencias y acciones manuales

---

## Referencias

- [GitHub Docs: Migrating from GitLab with GitHub Actions Importer](https://docs.github.com/en/actions/tutorials/migrate-to-github-actions/automated-migrations/gitlab-migration)
- [GitHub Docs: Migrating from GitLab CI/CD to GitHub Actions](https://docs.github.com/en/actions/migrating-to-github-actions/manually-migrating-to-github-actions/migrating-from-gitlab-cicd-to-github-actions)
- [GitHub Docs: Importing source code](https://docs.github.com/en/migrations/importing-source-code)
- [GitHub Docs: Extending Actions Importer with custom transformers](https://docs.github.com/en/actions/migrating-to-github-actions/automated-migrations/extending-github-actions-importer-with-custom-transformers)
- [GitHub Docs: Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions Importer (source)](https://github.com/github/gh-actions-importer)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
