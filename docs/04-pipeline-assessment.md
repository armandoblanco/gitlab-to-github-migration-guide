# Fase 4: Assessment de Pipelines

Antes de convertir un pipeline, es fundamental hacer un **assessment** (evaluación) para entender la complejidad, identificar riesgos y planificar la migración.

Existen dos herramientas complementarias:

| Herramienta | Tipo | Ideal para |
|---|---|---|
| **GitHub Actions Importer** (`gh actions-importer`) | CLI automatizada | Auditoría masiva de namespaces, forecast de uso, conversión automatizada |
| **Agente de GitHub Copilot** (`@gitlab-to-github`) | Asistente IA en VS Code | Análisis detallado pipeline por pipeline, recomendaciones contextuales |

---

## ¿Por qué hacer un assessment?

- Identificar features de GitLab que **no tienen equivalente** directo en GitHub Actions
- Detectar **secretos hardcodeados** en los pipelines
- Evaluar la **complejidad** de la migración (simple, media, alta)
- Identificar **dependencias externas** (registries, runners, servicios)
- Generar un plan de migración con esfuerzo estimado
- **Pronosticar** el uso de runners en GitHub Actions

---

## Assessment automatizado con GitHub Actions Importer

GitHub Actions Importer es la herramienta oficial de GitHub para migración automatizada de pipelines. Proporciona comandos específicos para cada fase del assessment.

> **Prerequisito:** Haber configurado las credenciales con `gh actions-importer configure` (ver [Prerrequisitos](01-prerequisites.md)).

### Paso 1: Auditoría (`audit`)

El comando `audit` analiza **todos los pipelines** de un namespace o grupo de GitLab y genera un reporte de compatibilidad.

```bash
# Auditar un namespace completo
gh actions-importer audit gitlab \
  --output-dir tmp/audit \
  --namespace mi-namespace-gitlab
```

Para auditar un namespace con subgrupos:

```bash
gh actions-importer audit gitlab \
  --output-dir tmp/audit \
  --namespace mi-namespace mi-namespace/subgrupo-uno mi-namespace/subgrupo-dos
```

> **Nota:** El comando `audit` requiere que las credenciales estén configuradas con una cuenta de **organización** de GitLab.

#### Interpretar el reporte de auditoría

El archivo `tmp/audit/audit_summary.md` contiene:

**Pipelines:**

| Resultado | Significado |
|---|---|
| Successful | 100% del pipeline convertido automáticamente |
| Partially successful | Estructura convertida, pero algunos items requieren ajuste manual |
| Unsupported | Tipo de pipeline no soportado por Actions Importer |
| Failed | Error fatal en la conversión (pipeline mal configurado o error interno) |

**Build steps:**

| Término | Significado |
|---|---|
| Known | Step convertido automáticamente a una action equivalente |
| Unknown | Step que no pudo ser convertido automáticamente |
| Unsupported | Step incompatible con GitHub Actions |
| Actions | Lista de actions usadas en los workflows convertidos |

**Tareas manuales:**

| Tarea | Acción requerida |
|---|---|
| Secrets | Crear manualmente en GitHub (no se migran valores) |
| Self-hosted runners | Configurar labels de runners en GitHub |
| Masked variables | Recrear como GitHub Secrets |
| Artifact reports | No soportados, requieren alternativa |

Adicionalmente, el archivo `workflow_usage.csv` lista todas las actions, secrets y runners por workflow convertido — útil para revisiones de seguridad.

### Paso 2: Forecast de uso (`forecast`)

El comando `forecast` analiza ejecuciones históricas de pipelines para pronosticar el uso de runners en GitHub Actions.

```bash
# Forecast de los últimos 7 días (default)
gh actions-importer forecast gitlab \
  --output-dir tmp/forecast \
  --namespace mi-namespace-gitlab
```

#### Interpretar el forecast

El archivo `tmp/forecast/forecast_report.md` incluye:

| Métrica | Descripción |
|---|---|
| **Job count** | Total de jobs completados en el período |
| **Pipeline count** | Pipelines únicos ejecutados |
| **Execution time** | Tiempo en runner (correlaciona con costo en GitHub-hosted runners) |
| **Queue time** | Tiempo esperando runner disponible |
| **Concurrent jobs** | Jobs ejecutándose simultáneamente (define cuántos runners necesitas) |

Estas métricas se desglosan por tipo de runner, útil si hay mezcla de hosted y self-hosted.

> Usa la [calculadora de precios de GitHub Actions](https://github.com/pricing/calculator) para estimar costos basándote en el execution time.

### Paso 3: Dry-run por proyecto

Para ver cómo quedaría un pipeline específico **sin hacer cambios**:

```bash
gh actions-importer dry-run gitlab \
  --output-dir tmp/dry-run \
  --namespace mi-namespace-gitlab \
  --project mi-proyecto
```

El directorio de salida contiene:
- El pipeline original
- El workflow convertido a GitHub Actions
- Logs de la conversión
- Stack traces si hubo errores

#### Usar un archivo local

Si prefieres convertir un `.gitlab-ci.yml` local:

```bash
gh actions-importer dry-run gitlab \
  --output-dir tmp/dry-run \
  --namespace mi-namespace-gitlab \
  --project mi-proyecto \
  --source-file-path ruta/al/.gitlab-ci.yml
```

---

## Assessment con GitHub Copilot (complemento)

El agente de Copilot es ideal para **análisis detallado** de un pipeline individual, obtener explicaciones contextuales y recomendaciones que van más allá de la conversión automática.

### Requisitos

- VS Code con **GitHub Copilot** habilitado
- El agente está disponible en: `.github/agents/gitlab-to-github.agents.md`

### Invocar el agente

1. Abrir VS Code
2. Abrir el chat de GitHub Copilot
3. Invocar el agente: **`@gitlab-to-github`**
4. Solicitar el assessment

### Prompt para assessment

```
@gitlab-to-github Analiza el siguiente pipeline de GitLab CI/CD y genera un reporte
de assessment para planificar la migración a GitHub Actions. No generes el workflow
todavía, solo el análisis.

[pegar o adjuntar el .gitlab-ci.yml]
```

### Qué incluye el assessment

El agente generará un reporte con:

#### 1. Inventario del pipeline

| Elemento | Cantidad | Detalle |
|---|---|---|
| Stages | N | Lista de stages |
| Jobs | N | Lista de jobs por stage |
| Variables | N | Variables de entorno definidas |
| Services | N | Contenedores de servicio |
| Includes/Extends | N | Templates y herencia |
| Artifacts/Cache | N | Configuración de persistencia |
| Environments | N | Ambientes de deploy |
| Rules/Conditions | N | Lógica condicional |

#### 2. Clasificación de complejidad

| Nivel | Criterio |
|---|---|
| **Simple** | Jobs con `script`, `image`, `variables`. Sin includes ni DAG complejo |
| **Media** | Usa `services`, `artifacts`, `cache`, `needs`, `rules` con condiciones |
| **Alta** | Usa `include`, `extends`, child pipelines, `parallel:matrix`, `resource_group`, `when:delayed` |

#### 3. Matriz de compatibilidad

| Feature de GitLab | Equivalente en GitHub Actions | Estado |
|---|---|---|
| `stages` + orden implícito | `jobs` con `needs` explícitos | Compatible |
| `services: postgres` | `services:` con health checks | Requiere ajustes |
| `include: template` | Expandir inline o reusable workflow | Manual |
| `when: delayed` | Sin equivalente | No soportado |
| `coverage:` regex | Sin equivalente nativo | Marketplace |

#### 4. Riesgos identificados

- Secretos hardcodeados en el pipeline
- Variables predefinidas de GitLab (`$CI_*`) que requieren mapeo
- Dependencias de runners específicos (`tags:`)
- Templates compartidos (`include:`) que necesitan resolverse

#### 5. Plan de migración

```
1. [ ] Resolver includes/extends (expandir inline)
2. [ ] Mapear variables $CI_* a contextos de GitHub
3. [ ] Convertir jobs con services (ajustar a localhost)
4. [ ] Configurar artifacts con upload/download-artifact
5. [ ] Migrar secrets a GitHub Secrets
6. [ ] Configurar environments con protection rules
7. [ ] Validar el workflow generado
```

---

## Assessment manual (sin Actions Importer ni Copilot)

Si no tienes acceso a GitHub Actions Importer ni a GitHub Copilot, usa este checklist:

### Checklist de evaluación

```bash
# 1. Contar jobs y stages
grep -c "stage:" .gitlab-ci.yml

# 2. Detectar includes
grep "include:" .gitlab-ci.yml

# 3. Detectar extends
grep "extends:" .gitlab-ci.yml

# 4. Detectar services
grep "services:" .gitlab-ci.yml

# 5. Detectar variables predefinidas de GitLab
grep -oE '\$CI_[A-Z_]+' .gitlab-ci.yml | sort -u

# 6. Detectar manual gates
grep "when: manual" .gitlab-ci.yml

# 7. Detectar features sin equivalente directo
grep -E "when: delayed|resource_group:|coverage:" .gitlab-ci.yml
```

### Clasificación rápida

| Si el pipeline tiene... | Complejidad |
|---|---|
| Solo `script`, `image`, `variables` | Simple |
| + `services`, `artifacts`, `cache`, `needs` | Media |
| + `include`, `extends`, `trigger`, `parallel:matrix` | Alta |

---

## Ejemplo de reporte de assessment

```markdown
## Assessment: proyecto-api/.gitlab-ci.yml

### Resumen
- **Complejidad**: Media
- **Stages**: 4 (build, test, quality, deploy)
- **Jobs**: 6
- **Variables CI_***: 8 (requieren mapeo)
- **Services**: 1 (PostgreSQL 15)
- **Secrets detectados**: 0

### Features compatibles
- stages → jobs con needs
- script → run
- image → container/setup-*
- variables → env
- needs → needs
- allow_failure → continue-on-error
- environment → environment

### Features que requieren ajustes
- services: postgres → services con ports y health checks (localhost en vez de hostname)
- artifacts:paths → actions/upload-artifact@v4 + download-artifact@v4
- cache: → actions/cache@v4 o cache integrado de setup-node
- when: manual → workflow_dispatch o environment con protection rules

### Features sin equivalente
- coverage: regex → no nativo, usar action del marketplace

### Esfuerzo estimado
- Conversión del pipeline: 1-2 horas
- Configuración de secrets en GitHub: 30 min
- Configuración de environments: 30 min
- Validación y ajustes: 1 hora
- **Total estimado**: ~3-4 horas
```

---

## Flujo recomendado de assessment

```
┌────────────────────────────────────────────┐
│  1. gh actions-importer audit              │  Visión macro: % de compatibilidad
│     Auditoría masiva del namespace         │  de todos los pipelines
└──────────────────┬─────────────────────────┘
                   │
┌──────────────────▼─────────────────────────┐
│  2. gh actions-importer forecast           │  Proyección de uso de runners
│     Pronóstico de runners y costos         │  y costos en GitHub Actions
└──────────────────┬─────────────────────────┘
                   │
┌──────────────────▼─────────────────────────┐
│  3. gh actions-importer dry-run            │  Conversión por proyecto
│     Dry-run por proyecto                   │  para ver el workflow resultante
└──────────────────┬─────────────────────────┘
                   │
┌──────────────────▼─────────────────────────┐
│  4. @gitlab-to-github (Copilot)            │  Análisis contextual de items
│     Revisar items parcialmente convertidos │  que requieren ajuste manual
└──────────────────┬─────────────────────────┘
                   │
┌──────────────────▼─────────────────────────┐
│  5. Migración                              │  Ejecutar con gh actions-importer
│     → Fase 5                               │  migrate o ajuste manual + Copilot
└────────────────────────────────────────────┘
```

---

## Siguiente paso

[Fase 5: Migración de Pipelines](05-pipeline-migration.md)
