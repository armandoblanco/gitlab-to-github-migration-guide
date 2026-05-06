# 🚀 Migración de GitLab a GitHub

## Introducción

Esta guía documenta el proceso completo para migrar **repositorios de código** y **pipelines CI/CD** desde GitLab hacia GitHub. Está diseñada para equipos de DevOps, desarrolladores y líderes técnicos que necesitan ejecutar migraciones de forma segura, repetible y con mínimo riesgo.

### ¿Qué cubre esta guía?

- **Migración de código fuente**: ramas, tags, historial de commits y archivos LFS
- **Escaneo y limpieza de secretos**: detección y remediación antes de hacer push a GitHub
- **Assessment de pipelines**: auditoría automatizada con [GitHub Actions Importer](https://docs.github.com/en/actions/tutorials/migrate-to-github-actions/automated-migrations/gitlab-migration) y análisis asistido con GitHub Copilot
- **Conversión de pipelines**: de `.gitlab-ci.yml` a GitHub Actions workflows
- **Post-migración**: seguridad, permisos, environments y cutover gradual

### ¿Qué herramientas se usan?

| Herramienta | Rol en la migración |
|---|---|
| **GitHub Actions Importer** | Auditoría, forecast y conversión automatizada de pipelines |
| **GitHub Copilot** (`@gitlab-to-github`) | Asistente IA para assessment detallado y ajustes de conversión |
| **gitleaks + BFG** | Escaneo y limpieza de secretos en la historia de commits |
| **GitHub CLI** | Autenticación, gestión de repositorios y operaciones de migración |

---

## 📋 Proceso de Migración

La migración se divide en 7 fases secuenciales. Haz clic en cualquier fase para ir a su documentación:

```mermaid
flowchart TD
    A["<b>Fase 1</b><br/>Prerrequisitos<br/><i>Herramientas, accesos y autenticación</i>"]
    B["<b>Fase 2</b><br/>Clonar Repositorio<br/><i>Clonar desde GitLab y preparar remotes</i>"]
    C["<b>Fase 3</b><br/>Escaneo de Secretos<br/><i>Detectar, limpiar y push a GitHub</i>"]
    D["<b>Fase 4</b><br/>Assessment de Pipelines<br/><i>Auditoría con Actions Importer + Copilot</i>"]
    E["<b>Fase 5</b><br/>Migración de Pipelines<br/><i>Conversión a GitHub Actions</i>"]
    F["<b>Fase 6</b><br/>Post-Migración<br/><i>Validación, seguridad y cutover</i>"]
    G["<b>Fase 7</b><br/>Troubleshooting<br/><i>Errores comunes y soluciones</i>"]

    A --> B --> C --> D --> E --> F
    F -.-> G

    style A fill:#2d333b,stroke:#58a6ff,color:#e6edf3
    style B fill:#2d333b,stroke:#58a6ff,color:#e6edf3
    style C fill:#2d333b,stroke:#d29922,color:#e6edf3
    style D fill:#2d333b,stroke:#a371f7,color:#e6edf3
    style E fill:#2d333b,stroke:#a371f7,color:#e6edf3
    style F fill:#2d333b,stroke:#3fb950,color:#e6edf3
    style G fill:#2d333b,stroke:#f85149,color:#e6edf3
```

### Flujo de herramientas para migración de pipelines

```mermaid
flowchart LR
    subgraph Assessment
        A1["<code>gh actions-importer audit</code><br/>Auditoría masiva"]
        A2["<code>gh actions-importer forecast</code><br/>Pronóstico de runners"]
        A3["<code>@gitlab-to-github</code><br/>Análisis con Copilot"]
    end

    subgraph Conversión
        B1["<code>gh actions-importer dry-run</code><br/>Preview local"]
        B2["<code>gh actions-importer migrate</code><br/>PR automático"]
        B3["<code>@gitlab-to-github</code><br/>Ajustes manuales"]
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

---

## ⚡ Quick Start

### Migrar un repositorio (código)

```bash
# 1. Clonar desde GitLab
git clone git@gitlab.com:GRUPO/REPO.git
cd REPO

# 2. Agregar remote de GitHub (NO hacer push todavía)
git remote add github git@github.com:ORG/REPO.git

# 3. Escanear secretos ANTES de hacer push
gitleaks detect --source . --verbose
# Si hay secretos → limpiar con BFG (ver docs/03-secret-scanning.md)

# 4. Push a GitHub (solo después de verificar que no hay secretos)
git push github --all
git push github --tags
```

### Migrar un pipeline (CI/CD) con GitHub Actions Importer

```bash
# 1. Instalar y configurar Actions Importer
gh extension install github/gh-actions-importer
gh actions-importer configure

# 2. Auditar el namespace completo
gh actions-importer audit gitlab --output-dir tmp/audit --namespace MI-NAMESPACE

# 3. Dry-run de un proyecto específico
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

## 📖 Documentación detallada

| # | Fase | Documento | Descripción |
|---|---|---|---|
| 1 | Prerrequisitos | [01-prerequisites.md](docs/01-prerequisites.md) | Herramientas, accesos, autenticación y consideraciones para entornos Enterprise (EMU, SAML SSO) |
| 2 | Clonar Repositorio | [02-repository-migration.md](docs/02-repository-migration.md) | Clonar desde GitLab, preparar remotes y métodos de migración |
| 3 | Escaneo y Push | [03-secret-scanning.md](docs/03-secret-scanning.md) | Escanear secretos, limpiar con BFG si es necesario, y push a GitHub |
| 4 | Assessment | [04-pipeline-assessment.md](docs/04-pipeline-assessment.md) | Auditoría automatizada con Actions Importer (`audit`, `forecast`) y análisis con Copilot |
| 5 | Migración de Pipelines | [05-pipeline-migration.md](docs/05-pipeline-migration.md) | Conversión con Actions Importer (`migrate`), ajustes con Copilot, tablas de equivalencias y ejemplos |
| 6 | Post-Migración | [06-post-migration.md](docs/06-post-migration.md) | Checklist de validación, seguridad, protección de ramas, environments y permisos |
| 7 | Troubleshooting | [07-troubleshooting.md](docs/07-troubleshooting.md) | Errores comunes: SAML SSO, push protection, refs rechazados, atribución de commits |

---

## 🤖 Agente de GitHub Copilot

Este repositorio incluye un **agente de GitHub Copilot** (`@gitlab-to-github`) especializado en migración de pipelines GitLab CI/CD → GitHub Actions. Funciona como **complemento** de GitHub Actions Importer para los ajustes que requieren contexto humano.

```mermaid
flowchart LR
    U["👤 Developer"] -->|"@gitlab-to-github"| C["🤖 Copilot Agent"]
    C -->|"Analiza"| GL[".gitlab-ci.yml"]
    C -->|"Genera"| GH[".github/workflows/*.yml"]
    C -->|"Reporta"| R["Advertencias<br/>Limitaciones<br/>Acciones manuales"]

    style U fill:#2d333b,stroke:#58a6ff,color:#e6edf3
    style C fill:#2d333b,stroke:#a371f7,color:#e6edf3
    style GL fill:#2d333b,stroke:#d29922,color:#e6edf3
    style GH fill:#2d333b,stroke:#3fb950,color:#e6edf3
    style R fill:#2d333b,stroke:#f85149,color:#e6edf3
```

### Ubicación

```
.github/agents/gitlab-to-github.agents.md
```

### Uso en VS Code

1. Abre VS Code con GitHub Copilot habilitado
2. En el chat de Copilot, invoca el agente: **`@gitlab-to-github`**
3. Adjunta o pega tu archivo `.gitlab-ci.yml`
4. El agente genera:
   - Workflow(s) de GitHub Actions equivalente(s)
   - Reporte de migración con advertencias y limitaciones
   - Instrucciones de implementación

### Capacidades del agente

- Transformación de stages, jobs, variables, rules, services, artifacts y cache
- Mapeo de variables predefinidas de GitLab (`$CI_*`) a contextos de GitHub
- Identificación de features sin equivalente directo
- Aplicación automática de mejores prácticas (versiones pinneadas, permisos mínimos, health checks)
- Reporte detallado de problemas, advertencias y acciones manuales

---

## 🔗 Referencias

- [GitHub Docs: Migrating from GitLab with GitHub Actions Importer](https://docs.github.com/en/actions/tutorials/migrate-to-github-actions/automated-migrations/gitlab-migration)
- [GitHub Docs: Migrating from GitLab CI/CD to GitHub Actions](https://docs.github.com/en/actions/migrating-to-github-actions/manually-migrating-to-github-actions/migrating-from-gitlab-cicd-to-github-actions)
- [GitHub Docs: Importing source code](https://docs.github.com/en/migrations/importing-source-code)
- [GitHub Docs: Extending Actions Importer with custom transformers](https://docs.github.com/en/actions/migrating-to-github-actions/automated-migrations/extending-github-actions-importer-with-custom-transformers)
- [GitHub Docs: Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions Importer (source)](https://github.com/github/gh-actions-importer)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)

---

## 📄 Licencia

Este proyecto es una guía de referencia para migración de GitLab a GitHub. Siéntete libre de adaptarlo a las necesidades de tu equipo.

---

<p align="center">
  <i>¿Encontraste un problema? Abre un <a href="../../issues">issue</a> o contribuye con un PR.</i>
</p>
