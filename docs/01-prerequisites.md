# Fase 1: Prerrequisitos

## Herramientas necesarias

| Herramienta | Propósito | Instalación (macOS) |
|---|---|---|
| Git | Control de versiones | `brew install git` |
| GitHub CLI | Autenticación y gestión de repos | `brew install gh` |
| GitHub Actions Importer | Migración automatizada de pipelines | `gh extension install github/gh-actions-importer` |
| Docker | Requerido por Actions Importer | [Docker Desktop](https://docs.docker.com/get-docker/) |
| gitleaks | Escaneo de secretos en la historia | `brew install gitleaks` |
| BFG Repo-Cleaner | Limpieza de secretos en la historia | `brew install bfg` |
| jq | Procesamiento de JSON | `brew install jq` |

> **Nota:** GitHub Actions Importer se ejecuta en un contenedor Docker. No necesita estar en el mismo servidor que GitLab.

### Verificar instalación

```bash
git --version
gh --version
gh actions-importer version
docker --version
gitleaks version
jq --version
```

---

## Accesos requeridos

### En GitLab (origen)

- [ ] Acceso de **lectura** al repositorio (SSH o HTTPS)
- [ ] **Token de acceso personal** de GitLab con scope `read_api` (requerido para Actions Importer)

### En GitHub (destino)

- [ ] Cuenta de usuario con acceso a la organización destino
- [ ] **Permisos de escritura** en la organización donde se creará el repositorio
- [ ] **Personal Access Token (classic)** con scope `workflow` (requerido para Actions Importer)
- [ ] SSH key configurada (para push de código)

### Configurar GitHub Actions Importer

```bash
# Configuración interactiva (guarda credenciales en variables de entorno)
gh actions-importer configure
```

El comando solicitará:
1. **CI provider**: Seleccionar `GitLab`
2. **Personal access token for GitHub**: PAT con scope `workflow`
3. **Base url of GitHub**: `https://github.com` (o tu instancia Enterprise)
4. **Private token for GitLab**: PAT de GitLab con scope `read_api`
5. **Base url of GitLab**: URL de tu instancia GitLab

```bash
# Actualizar el contenedor a la última versión
gh actions-importer update
```

---

## Autenticación en GitHub

### Verificar autenticación

```bash
# Con GitHub CLI
gh auth status

# Con SSH
ssh -T git@github.com
```

### Autenticarse

```bash
# Autenticación interactiva
gh auth login
```

### Organizaciones con SAML SSO

Si la organización destino tiene **SAML SSO** habilitado, la SSH key o PAT debe ser autorizada:

1. Ir a **https://github.com/settings/keys**
2. Buscar la SSH key en la lista
3. Clic en **"Configure SSO"** al lado de la key
4. Seleccionar la organización y clic en **"Authorize"**
5. Completar la autenticación SAML

> Si usas PAT en vez de SSH, el proceso es el mismo desde `Settings > Developer settings > Personal access tokens`.

---

## Consideraciones para GitHub Enterprise (EMU)

Si la organización destino es parte de una **GitHub Enterprise con Enterprise Managed Users (EMU)**, ten en cuenta:

### Restricciones de visibilidad

| Característica | Comportamiento en EMU |
|---|---|
| Repositorios públicos | **NO permitidos** — solo privados e internos |
| Interacción fuera de la Enterprise | **NO permitida** |
| Forks | Solo dentro de la Enterprise |
| Gists | **NO disponibles** |

### Autenticación y acceso

- Los usuarios se autentican mediante el **IdP** (SAML o OIDC)
- Los usernames son asignados por el IdP con formato `SHORTCODE_usuario`
- Los PAT deben ser autorizados para SSO
- Si la Enterprise tiene **IP Allow List**, verificar que tu IP esté permitida

### IdP Providers soportados

| IdP | SAML | OIDC | SCIM |
|---|---|---|---|
| Microsoft Entra ID | Si | Si | Si |
| Okta | Si | No | Si |
| PingFederate | Si | No | Si |

### Estructura organizacional

```
Enterprise
├── Organización A
│   ├── repo-1 (privado)
│   └── repo-2 (interno)
├── Organización B
│   └── repo-3 (privado)
└── Usuarios EMU (provisioned via IdP)
```

> **Interno** vs **Privado**: Los repositorios **internos** son visibles para todos los miembros de la Enterprise. Los **privados** solo para quienes tienen acceso explícito.

### Múltiples cuentas (personal + EMU)

```bash
# ~/.gitconfig — usar diferentes identidades por directorio
[includeIf "gitdir:~/trabajo/"]
    path = ~/.gitconfig-empresa

[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

```bash
# ~/.gitconfig-empresa
[user]
    name = "Tu Nombre"
    email = "tu-email@empresa.com"

# ~/.gitconfig-personal
[user]
    name = "Tu Nombre"
    email = "tu-email@personal.com"
```

---

## Siguiente paso

[Fase 2: Clonar y Preparar el Repositorio](02-repository-migration.md)
