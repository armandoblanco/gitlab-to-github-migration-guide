# Fase 3: Escaneo de Secretos y Push a GitHub

GitHub **Push Protection** bloquea automáticamente los push que contengan secretos detectados (tokens, API keys, passwords) en cualquier commit de la historia. Por eso, el escaneo y limpieza se realiza **antes** de hacer push a GitHub.

---

## Paso 1: Escanear el repositorio

```bash
cd mi-repo

# Escaneo verbose en consola
gitleaks detect --source . --verbose

# Generar reporte en JSON para procesamiento
gitleaks detect --source . --report-format json --report-path leaks.json
```

### Interpretar el reporte

El reporte muestra para cada secreto:
- **Tipo** de secreto (GitLab Access Token, AWS Key, etc.)
- **Commit** donde se introdujo
- **Archivo y línea** donde está el secreto
- **El secreto** en sí mismo

### Verificar un secreto específico

```bash
# Ver el contexto del secreto en un commit
git show COMMIT_HASH:ruta/al/archivo | sed -n '29,33p'
```

### Si NO hay secretos

El repositorio está limpio. Ir directamente al [Paso 7: Push a GitHub](#paso-7-push-a-github).

### Si HAY secretos

Continuar con el Paso 2.

---

## Paso 2: Revocar los secretos en GitLab

**Antes de limpiar**, revocar todos los tokens y secretos detectados en el sistema de origen. Limpiar la historia **no invalida** los tokens — siguen siendo funcionales hasta que se revoquen manualmente.

---

## Paso 3: Generar archivo de secretos a limpiar

```bash
# Extraer secretos únicos del reporte JSON
jq -r '.[].Secret' leaks.json | sort -u > secrets.txt

# Verificar el contenido
cat secrets.txt
```

---

## Paso 4: Limpiar la historia con BFG

```bash
# Ejecutar BFG (reemplaza los secretos por ***REMOVED***)
bfg --replace-text secrets.txt

# Limpiar refs y garbage collect
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

---

## Paso 5: Verificar la limpieza

```bash
# Re-escanear — no debe encontrar secretos
gitleaks detect --source . --verbose
```

Si `gitleaks` sigue encontrando secretos, repetir el proceso agregando los faltantes a `secrets.txt`.

---

## Paso 6: Limpiar archivos temporales

```bash
rm secrets.txt leaks.json
```

---

## Paso 7: Push a GitHub

Una vez verificado que el repositorio está limpio de secretos, hacer push:

```bash
# Push todas las ramas y tags
git push github --all
git push github --tags
```

Si se limpió la historia con BFG y ya existía un push previo, usar `--force`:

```bash
git push github --all --force
git push github --tags --force
```

### Si el repositorio usa Git LFS

```bash
# Push objetos LFS al nuevo remote
git lfs push --all github
```

> **Importante:** Si se reescribió la historia con BFG, cualquier persona con un clon local deberá hacer `git clone` nuevamente.

### Verificar el push

```bash
# Verificar ramas en GitHub
git branch -r --list 'github/*'

# Verificar tags
git ls-remote --tags github

# Abrir en el navegador
gh repo view ORG/REPO --web
```

---

## Alternativa: Permitir el secreto en GitHub

Si el secreto ya fue revocado y prefieres no reescribir la historia, GitHub proporciona una URL para desbloquear el push:

```
https://github.com/ORG/REPO/security/secret-scanning/unblock-secret/HASH
```

Esta URL aparece en el mensaje de error del push. Puedes marcar el secreto como falso positivo o indicar que ya fue revocado.

---

## Recomendación para migraciones masivas

Antes de migrar cada repositorio, ejecutar el escaneo como parte del proceso estándar:

```bash
# Script de pre-migración
cd $REPO
gitleaks detect --source . --report-format json --report-path leaks.json

LEAKS=$(jq length leaks.json)
if [ "$LEAKS" -gt 0 ]; then
    echo "ADVERTENCIA: Se encontraron $LEAKS secretos. Limpiar antes de hacer push."
    jq -r '.[] | "  - \(.RuleID): \(.File):\(.StartLine)"' leaks.json
else
    echo "OK: Sin secretos detectados. Listo para push a GitHub."
    git push github --all
    git push github --tags
fi
```

---

## Siguiente paso

[Fase 4: Assessment de Pipelines](04-pipeline-assessment.md)
