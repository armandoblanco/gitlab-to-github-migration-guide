# Fase 7: Troubleshooting

## Errores comunes durante la migración

---

### Error: SAML SSO — push denegado

**Síntoma:**
```
ERROR: Permission to ORG/REPO.git denied
fatal: Could not read from remote repository.
```

**Causa:** La organización tiene SAML SSO habilitado y la SSH key / PAT no ha sido autorizada.

**Solución (SSH):**
1. Ir a `GitHub.com > Settings > SSH and GPG keys`
2. Encontrar la clave SSH
3. Clic en **Configure SSO** junto a la clave
4. Autorizar la organización

**Solución (PAT):**
1. Ir a `GitHub.com > Settings > Developer settings > Personal access tokens`
2. Encontrar el token
3. Clic en **Configure SSO**
4. Autorizar la organización

---

### Error: Push Protection — secreto detectado (GH013)

**Síntoma:**
```
remote: error: GH013: Repository rule violations found
remote: - GITHUB PUSH PROTECTION
remote:   — Push cannot contain secrets
```

**Causa:** Un commit en la historia contiene un secreto (token, API key, password).

**Solución A — Limpiar la historia:**
```bash
# Ver qué secreto se detectó
gitleaks detect --source . --verbose

# Generar archivo de secretos
gitleaks detect --source . --report-format json --report-path leaks.json
jq -r '.[].Secret' leaks.json | sort -u > secrets.txt

# Limpiar con BFG
bfg --replace-text secrets.txt
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Re-escanear
gitleaks detect --source . --verbose

# Push forzado
git push github --all --force
git push github --tags --force
```

**Solución B — Desbloquear en GitHub:**
El mensaje de error incluye una URL para permitir el push:
```
https://github.com/ORG/REPO/security/secret-scanning/unblock-secret/HASH
```
Marcar como revocado o falso positivo.

Ver detalles en [Fase 3: Escaneo de Secretos](03-secret-scanning.md).

---

### Error: Repository not found

**Síntoma:**
```
remote: Repository not found.
fatal: repository 'https://github.com/ORG/REPO/' not found
```

**Causas posibles:**
1. **URL incorrecta**: Verificar que no hay trailing slash o typo
2. **Repositorio no existe**: Crearlo primero en GitHub
3. **Sin permisos**: Verificar acceso al repo
4. **PAT sin scope**: Necesita `repo` scope
5. **SSO no autorizado**: Autorizar el PAT para la organización

**Diagnóstico:**
```bash
# Verificar URL del remote
git remote -v

# Verificar acceso
gh repo view ORG/REPO

# Verificar autenticación
gh auth status
```

---

### Error: Push al remote equivocado

**Síntoma:**
```bash
git push --mirror
# ¡Se hizo push a GitLab (origin) en vez de GitHub!
```

**Causa:** `git push --mirror` sin especificar remote usa `origin`, que apunta a GitLab.

**Solución:**
```bash
# Siempre especificar el remote
git push github --all
git push github --tags

# Verificar remotes configurados
git remote -v
```

**Prevención:** Usar `--all` + `--tags` en vez de `--mirror` para evitar confusiones.

---

### Error: Push rechazado por tamaño

**Síntoma:**
```
remote: error: File ARCHIVO is 123.45 MB; this exceeds GitHub's file size limit of 100.00 MB
```

**Solución:**
```bash
# Opción 1: Configurar Git LFS
git lfs install
git lfs track "*.bin" "*.zip" "*.tar.gz"
git add .gitattributes
git commit -m "Configure Git LFS"

# Opción 2: Limpiar archivos grandes de la historia
bfg --strip-blobs-bigger-than 100M
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

---

### Error: Refs rechazados en mirror

**Síntoma:**
```
! [remote rejected] refs/keep-around/... (deny updating a hidden ref)
! [remote rejected] refs/merge-requests/... (deny updating a hidden ref)
! [remote rejected] refs/pipelines/... (deny updating a hidden ref)
```

**Causa:** `--mirror` intenta pushear refs internos de GitLab que GitHub no acepta.

**Solución:** Usar `--all` + `--tags` en vez de `--mirror`:
```bash
git push github --all
git push github --tags
```

---

### Error: Workflow no ejecuta

**Síntomas:**
- El archivo `.yml` está en `.github/workflows/` pero no aparece en la pestaña Actions
- El workflow aparece pero no se dispara

**Causas y soluciones:**

| Causa | Solución |
|---|---|
| YAML inválido | Validar con `actionlint` o el editor |
| Trigger incorrecto | Verificar sección `on:` |
| Workflow en rama no default | Hacer merge a `main` primero |
| Actions deshabilitados | `Settings > Actions > General > Allow all actions` |
| Permisos insuficientes | `Settings > Actions > General > Workflow permissions` |

---

### Error: Service no accesible (conexión rechazada)

**Síntoma:**
```
Error: connect ECONNREFUSED postgres:5432
```

**Causa:** En GitHub Actions, los servicios están en `localhost`, no en el hostname del contenedor.

**Solución:**
```yaml
# Incorrecto (funciona en GitLab)
DATABASE_URL: postgres://user:pass@postgres:5432/db

# Correcto (GitHub Actions)
DATABASE_URL: postgres://user:pass@localhost:5432/db
```

Agregar health checks para esperar que el servicio esté listo:
```yaml
services:
  postgres:
    image: postgres:15
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

---

### Error: Secrets no disponibles en fork PRs

**Síntoma:** El workflow falla en PRs de forks porque `${{ secrets.MI_SECRET }}` está vacío.

**Causa:** Por seguridad, GitHub no expone secrets de la organización a workflows ejecutados desde forks.

**Soluciones:**
1. Usar `pull_request_target` (con precaución)
2. Separar el workflow en dos: build en `pull_request`, deploy en `pull_request_target`
3. Para PRs de forks, usar solo `GITHUB_TOKEN` (disponible siempre)

---

### Error: Atribución de commits incorrecta

**Síntoma:** Los commits aparecen con autor incorrecto o como "ghost user".

**Causa:** El email del commit no coincide con ninguna cuenta de GitHub.

**Solución:**
```bash
# Verificar emails en los commits
git log --format="%ae" | sort -u

# Mapear con .mailmap
echo "Nombre <email-github@example.com> <email-gitlab@example.com>" >> .mailmap
```

Cada desarrollador debe agregar su email de GitLab en GitHub:
```
Settings > Emails > Add email address
```

---

### Error: Git LFS — objetos faltantes

**Síntoma:**
```
Uploading LFS objects: ... error: ... does not exist
```

**Solución:**
```bash
# Descargar todos los objetos LFS del origen
git lfs fetch --all origin

# Push al nuevo remote
git lfs push --all github
```

---

## Verificación final

Después de resolver todos los problemas, ejecutar esta verificación:

```bash
echo "=== Verificación de migración ==="

# 1. Ramas
echo "Ramas en GitHub:"
git branch -r | grep github | wc -l

# 2. Tags
echo "Tags:"
git tag -l | wc -l

# 3. Último commit
echo "Último commit en main:"
git log github/main --oneline -1

# 4. Remotes
echo "Remotes:"
git remote -v

# 5. LFS (si aplica)
echo "Archivos LFS:"
git lfs ls-files | wc -l

echo "=== Verificación completa ==="
```

---

## ¿Necesitas ayuda?

Si encuentras un problema no documentado aquí:

1. Usa el agente de Copilot: `@gitlab-to-github describe el error y pide ayuda`
2. Consulta la [documentación de GitHub](https://docs.github.com/en/migrations)
3. Abre un issue en este repositorio

---

## Navegación

| Anterior | Inicio |
|---|---|
| [Fase 6: Post-Migración](06-post-migration.md) | [README](../README.md) |
