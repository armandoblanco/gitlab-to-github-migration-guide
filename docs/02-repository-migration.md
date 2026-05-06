# Fase 2: Clonar y Preparar el Repositorio

En esta fase se clona el repositorio desde GitLab y se preparan los remotes. **El push a GitHub se realiza después del escaneo de secretos** (Fase 3).

## Preparar el repositorio destino en GitHub

Crear un repositorio **vacío** en la organización destino:

1. Ve a la organización: `https://github.com/orgs/ORG/repositories`
2. Clic en **New repository**
3. Selecciona la **organización** como Owner
4. Ingresa el nombre del repositorio
5. **NO** inicializar con README, .gitignore ni licencia
6. Clic en **Create repository**

> El repositorio debe estar completamente vacío para evitar conflictos durante el push.

---

## Método A: Línea de Comandos (Recomendado)

Preserva todo el historial de commits, ramas y tags.

### Si ya tienes un clon local

```bash
cd mi-repo

# Verificar remotes actuales
git remote -v

# Agregar GitHub como nuevo remote
git remote add github git@github.com:ORG/REPO.git

# Verificar
git remote -v
```

> **⚠️ No hacer push todavía.** Primero completar el [escaneo de secretos](03-secret-scanning.md).

### Si necesitas clonar desde GitLab

```bash
# Clonar el repositorio
git clone git@gitlab.com:GRUPO/PROYECTO/REPO.git
cd REPO

# Agregar remote de GitHub
git remote add github git@github.com:ORG/REPO.git
```

> **⚠️ No hacer push todavía.** Primero completar el [escaneo de secretos](03-secret-scanning.md).

---

## Método B: GitHub Importer (Web UI)

Ideal para migraciones simples desde la interfaz web.

1. Inicia sesión en GitHub
2. Clic en **+** → **Import repository**
3. En **"Your old repository's clone URL"**, ingresar:
   ```
   https://gitlab.com/USUARIO/REPO.git
   ```
4. Si el repo es privado, ingresar credenciales de GitLab
5. Elegir la organización como propietario
6. Ingresar el nombre del repositorio
7. Clic en **Begin import**

**Limitaciones:**
- Solo importa código fuente e historial de commits
- NO migra issues, merge requests ni CI/CD
- NO migra objetos de Git LFS
- El repositorio de GitLab debe ser accesible desde internet

> **⚠️ Nota sobre secretos:** GitHub Importer importa directamente sin pasar por escaneo local. Si Push Protection detecta secretos, la importación puede fallar. Para repositorios con posibles secretos en la historia, usar el **Método A** (CLI) que permite escanear antes de hacer push.

---

## Método C: GitHub CLI

```bash
# Autenticarse
gh auth login

# Clonar desde GitLab como bare
git clone --bare https://gitlab.com/USUARIO/REPO.git
cd REPO.git

# Crear repo en GitHub (sin push)
gh repo create ORG/REPO --private
```

> **⚠️ No hacer push todavía.** Para repositorios con `--bare`, escanear secretos antes de continuar. Ver [Fase 3](03-secret-scanning.md).

---

## `--all` + `--tags` vs `--mirror`

| | `--all` + `--tags` | `--mirror` |
|---|---|---|
| Ramas y tags | ✅ Sí | ✅ Sí |
| Refs internas de GitLab (MRs) | ❌ No (no se necesitan) | ✅ Sí (pueden causar errores) |
| Elimina en destino lo que no exista en origen | ❌ No | ⚠️ Sí |
| Riesgo | Bajo | Alto |

**Recomendación:** Usar `--all` + `--tags`. El modo `--mirror` puede fallar con refs internas de GitLab que GitHub rechaza.

---

## Match de usuarios entre GitLab y GitHub

### Comportamiento

Git **no hace match automático** de usuarios. La historia se sube con el `name` y `email` del autor original.

| Escenario | Resultado en GitHub |
|---|---|
| El email del commit coincide con un email verificado en GitHub | Commit linkeado al perfil |
| El email no coincide | Nombre como texto plano, sin link |

### Solución recomendada

Cada usuario debe agregar su email de GitLab a su cuenta de GitHub:

**GitHub → Settings → Emails → Add email address**

Esto linkea automáticamente todos los commits históricos al perfil correcto sin modificar la historia.

---

## Migración de Git LFS

Si el repositorio usa Git LFS, descargar los objetos **antes** del push:

```bash
# En el clone local desde GitLab
git lfs fetch --all origin
```

> El push de objetos LFS se hará junto con el push de código en la [Fase 3](03-secret-scanning.md), después de escanear secretos.

---

## Verificación

Después de completar la Fase 3 (escaneo + push), verificar:

```bash
# Verificar ramas
git branch -r --list 'github/*'

# Verificar tags
git ls-remote --tags github

# Verificar en browser
gh repo view ORG/REPO --web
```

### Checklist

- [ ] Todas las ramas están presentes
- [ ] Todos los tags están presentes
- [ ] La historia de commits está intacta
- [ ] Los archivos del HEAD coinciden con el original

---

## Siguiente paso

➡️ [Fase 3: Escaneo de Secretos y Push a GitHub](03-secret-scanning.md)
