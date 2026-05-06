# Guía de Migración de Repositorios: GitLab → GitHub

## Índice

1. [Pre-requisitos](#1-pre-requisitos)
2. [Autenticación en GitHub](#2-autenticación-en-github)
3. [Preparar el repositorio destino en GitHub](#3-preparar-el-repositorio-destino-en-github)
4. [Clonar el repositorio de GitLab](#4-clonar-el-repositorio-de-gitlab)
5. [Escanear secretos antes de migrar](#5-escanear-secretos-antes-de-migrar)
6. [Limpiar secretos de la historia](#6-limpiar-secretos-de-la-historia)
7. [Push al repositorio en GitHub](#7-push-al-repositorio-en-github)
8. [Verificación post-migración](#8-verificación-post-migración)
9. [Match de usuarios entre GitLab y GitHub](#9-match-de-usuarios-entre-gitlab-y-github)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Pre-requisitos

### Herramientas necesarias

```bash
# Git
git --version

# GitHub CLI
brew install gh

# Escáner de secretos
brew install gitleaks

# Limpiador de historia
brew install bfg

# Procesador JSON (para extraer secretos del reporte)
brew install jq
```

### Accesos requeridos

- Acceso de lectura al repositorio en GitLab (SSH o HTTPS)
- Acceso de escritura a la organización destino en GitHub
- Si la organización tiene **SAML SSO** habilitado, la llave SSH o PAT debe estar autorizada (ver sección de Troubleshooting)

---

## 2. Autenticación en GitHub

### Verificar autenticación

```bash
# Con GitHub CLI
gh auth status

# Con SSH
ssh -T git@github.com
```

### Autenticarse si es necesario

```bash
gh auth login
```

### Si la organización tiene SAML SSO

1. Ir a **https://github.com/settings/keys**
2. Buscar la SSH key en la lista
3. Clic en **"Configure SSO"** al lado de la key
4. Seleccionar la organización y clic en **"Authorize"**
5. Completar la autenticación SAML

---

## 3. Preparar el repositorio destino en GitHub

Crear un repositorio **vacío** en la organización destino:

- **NO** inicializar con README
- **NO** agregar .gitignore
- **NO** agregar licencia

El repositorio debe estar completamente vacío para evitar conflictos.

---

## 4. Clonar el repositorio de GitLab

### Opción A: Si ya tienes un clon local

```bash
cd mi-repo

# Verificar remotes actuales
git remote -v

# Agregar GitHub como nuevo remote
git remote add github git@github.com:ORG/REPO.git

# Verificar que se agregó correctamente
git remote -v
```

### Opción B: Clonar desde GitLab para migrar

```bash
# Clonar el repositorio
git clone git@gitlab.com:GRUPO/PROYECTO/REPO.git
cd REPO

# Agregar remote de GitHub
git remote add github git@github.com:ORG/REPO.git
```

> **Nota:** Reemplazar `ORG` con el nombre de la organización en GitHub y `REPO` con el nombre del repositorio.

---

## 5. Escanear secretos antes de migrar

**Este paso es crítico.** GitHub Push Protection bloqueará el push si detecta secretos en la historia de commits.

### Ejecutar escaneo

```bash
# Escaneo con reporte verbose en consola
gitleaks detect --source . --verbose

# Generar reporte en JSON para procesamiento posterior
gitleaks detect --source . --report-format json --report-path leaks.json
```

### Interpretar el reporte

El reporte mostrará:
- **Tipo de secreto** detectado (tokens, passwords, API keys)
- **Commit** donde se introdujo
- **Archivo y línea** donde está el secreto
- **El secreto** en sí mismo

### Verificar un secreto específico en un commit

```bash
git show COMMIT_HASH:ruta/al/archivo | sed -n 'LINEA_INICIO,LINEA_FINp'
```

### Si NO hay secretos

Ir directamente al [Paso 7: Push al repositorio en GitHub](#7-push-al-repositorio-en-github).

### Si HAY secretos

Continuar con el [Paso 6: Limpiar secretos de la historia](#6-limpiar-secretos-de-la-historia).

---

## 6. Limpiar secretos de la historia

### 6.1 Revocar los secretos en GitLab

**Antes de limpiar**, revocar todos los tokens y secretos detectados en GitLab. Limpiar la historia no invalida los tokens — siguen siendo funcionales hasta que se revoquen.

### 6.2 Generar archivo de secretos a limpiar

```bash
# Extraer secretos únicos del reporte JSON de gitleaks
jq -r '.[].Secret' leaks.json | sort -u > secrets.txt

# Verificar el contenido
cat secrets.txt
```

### 6.3 Limpiar la historia con BFG

```bash
# Ejecutar BFG (reemplaza los secretos por ***REMOVED***)
bfg --replace-text secrets.txt

# Limpiar refs y garbage collect
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

### 6.4 Verificar que la limpieza fue exitosa

```bash
# Re-escanear — no debe encontrar secretos
gitleaks detect --source . --verbose
```

Si `gitleaks` sigue encontrando secretos, repetir el proceso agregando los faltantes a `secrets.txt`.

### 6.5 Eliminar archivo de secretos

```bash
rm secrets.txt leaks.json
```

> **Importante:** BFG reescribe los commits, por lo que los hashes cambian. Cualquier persona con un clon local deberá hacer `git clone` nuevamente después de la migración.

---

## 7. Push al repositorio en GitHub

### Push recomendado (todas las ramas + tags)

```bash
git push github --all
git push github --tags
```

### Si se limpió la historia (paso 6), usar --force

```bash
git push github --all --force
git push github --tags --force
```

### ¿Por qué NO usar `--mirror`?

| | `--all` + `--tags` | `--mirror` |
|---|---|---|
| Ramas y tags | ✅ Sí | ✅ Sí |
| Refs internas de GitLab (MRs, etc.) | ❌ No (no se necesitan) | ✅ Sí (pueden causar errores) |
| Elimina en destino lo que no exista en origen | ❌ No | ⚠️ Sí |
| Riesgo | Bajo | Alto |

`--mirror` puede fallar con refs internas de GitLab que GitHub rechaza. Se recomienda `--all` + `--tags` para migraciones.

---

## 8. Verificación post-migración

```bash
# Verificar ramas subidas
git branch -r --list 'github/*'

# Verificar tags
git ls-remote --tags github

# Verificar en GitHub CLI
gh repo view ORG/REPO --web
```

### Checklist de verificación

- [ ] Todas las ramas están presentes en GitHub
- [ ] Todos los tags están presentes
- [ ] La historia de commits está intacta
- [ ] Los archivos del HEAD coinciden con el original
- [ ] No hay alertas de secretos en GitHub (Security → Secret scanning)

---

## 9. Match de usuarios entre GitLab y GitHub

### Comportamiento por defecto

Git **no hace match automático** de usuarios. La historia se sube con el `name` y `email` del autor original de GitLab.

| Escenario | Resultado en GitHub |
|---|---|
| El email del commit coincide con un email verificado en GitHub | El commit se linkea al perfil de GitHub |
| El email no coincide | El nombre aparece como texto plano sin link |

### Solución recomendada

Cada usuario debe agregar su email corporativo (el que usaba en GitLab) a su cuenta de GitHub:

**GitHub → Settings → Emails → Add email address**

Esto linkea automáticamente todos los commits históricos al perfil correcto.

### Alternativa: Reescribir la historia (no recomendado)

```bash
# Crear archivo de mapeo
cat > mailmap.txt << 'EOF'
Nombre Nuevo <nuevo@email.com> <viejo@email-gitlab.com>
EOF

# Reescribir
git filter-repo --mailmap mailmap.txt
```

> **No recomendado** porque cambia los hashes de todos los commits y rompe referencias existentes.

---

## 10. Troubleshooting

### Error: SAML SSO

```
ERROR: The 'ORG' organization has enabled or enforced SAML SSO.
```

**Solución:** Autorizar la SSH key o PAT para la organización con SSO.
Ver [Paso 2: Autenticación en GitHub](#2-autenticación-en-github).

### Error: Push Protection (secretos detectados)

```
error: GH013: Repository rule violations found for refs/heads/main.
- GITHUB PUSH PROTECTION - Push cannot contain secrets
```

**Solución:** Limpiar secretos antes de migrar.
Ver [Paso 5](#5-escanear-secretos-antes-de-migrar) y [Paso 6](#6-limpiar-secretos-de-la-historia).

### Error: Repository not found

```
fatal: repository 'https://github.com/ORG/REPO/' not found
```

**Posibles causas:**
- El repositorio no existe en GitHub (crearlo primero)
- No tienes permisos de escritura
- La URL tiene un trailing slash extra
- No estás autenticado

**Verificar:**
```bash
gh auth status
git remote -v
```

### Error: Remote rejected (deny updating a hidden ref)

```
! [remote rejected] origin/HEAD -> origin/HEAD (deny updating a hidden ref)
```

**Causa:** Se usó `git push --mirror` sin especificar el remote, y se intentó pushear refs internas a GitLab.

**Solución:** Usar `git push github --all` en vez de `--mirror`.

### Error: Push to wrong remote

Si `git push --mirror` pushea a GitLab en vez de GitHub, es porque el remote por defecto (`origin`) sigue apuntando a GitLab.

**Solución:** Especificar siempre el remote:
```bash
git push github --all
# NO: git push --all (usa origin por defecto)
```

---

## Resumen del proceso

```
┌─────────────────────────────┐
│  1. Verificar autenticación │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│  2. Crear repo vacío en GH  │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│  3. Clonar / agregar remote │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│  4. Escanear con gitleaks   │
└──────────────┬──────────────┘
               │
        ┌──────▼──────┐
        │ ¿Secretos?  │
        └──────┬──────┘
         Sí    │    No
    ┌──────────┤    │
    │          │    │
┌───▼────┐     │    │
│ Limpiar│     │    │
│ con BFG│     │    │
└───┬────┘     │    │
    │          │    │
    └──────────┘    │
               │    │
┌──────────────▼────▼─────────┐
│  5. git push github --all   │
│     git push github --tags  │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│  6. Verificar en GitHub     │
└─────────────────────────────┘
```
