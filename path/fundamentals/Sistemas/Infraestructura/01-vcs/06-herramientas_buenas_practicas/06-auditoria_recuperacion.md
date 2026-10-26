# Auditoria, recuperacion y limpieza

Git almacena todo el historial de cambios como objetos inmutables, lo que significa que nada se pierde realmente: cada commit, cada rama eliminada, cada cambio sobrescrito puede recuperarse. Esta propiedad convierte a Git en una herramienta de auditoria poderosa, pero tambien requiere herramientas para limpiar objetos que ya no se necesitan y para detectar informacion sensible que pueda haber sido comprometida accidentalmente.

## Historial de referencia (reflog)

El reflog es la "red de seguridad" de Git. Registra cada movimiento de HEAD y de las ramas, incluyendo commits que ya no estan en ninguna rama (commits "huérfanos").

```bash
# Ver el historial de referencia
git reflog
# a1b2c3d HEAD@{0}: checkout: moving from feature to main
# e4f5g6h HEAD@{1}: commit: Agregar busqueda avanzada
# i7j8k9l HEAD@{2}: reset: moving to HEAD~3

# Recuperar un commit "perdido"
git checkout e4f5g6h

# Recuperar una rama eliminada
git branch mi-rama-eliminada e4f5g6h

# Recuperar un stash eliminado
git stash list    # No se puede recuperar si se dio drop
```

**Ejemplo comun:** eliminaste una rama por error y necesitas recuperarla.

```bash
# Buscar el ultimo commit de la rama en el reflog
git reflog
# a1b2c3d HEAD@{5}: checkout: moving from feature-busqueda to main

# Recrear la rama
git branch feature-busqueda a1b2c3d
```

## Recuperacion de cambios

```bash
# Deshacer el ultimo commit pero conservar los cambios
git reset --soft HEAD~1

# Deshacer el ultimo commit y los cambios en archivos
git reset --hard HEAD~1

# Recuperar un archivo especifico de un commit anterior
git checkout a1b2c3d -- archivo.py

# Recuperar todo el repositorio a un estado anterior
git reset --hard a1b2c3d

# Crear un commit que deshace los cambios de otro commit
git revert a1b2c3d
```

> **Diferencia critica:** `reset` mueve el puntero de la rama (reescribe historial); `revert` crea un nuevo commit que invierte los cambios (conserva historial). Usa `revert` para cambios ya compartidos con el equipo.

## Recoleccion de basura (garbage collection)

Git almacena objetos eliminados durante un tiempo por si necesitan recuperarse, pero eventualmente los limpia para optimizar espacio.

```bash
# Ejecutar limpieza manual
git gc

# Limpiar agresivamente (objets inaccesibles)
git gc --aggressive --prune=now

# Ver espacio recuperable
git count-objects -v
```

**Cuando ejecutar gc:**
- Despues de hacer rebase o reset sobre historial largo.
- Cuando el repositorio ocupa demasiado espacio.
- Despues de eliminar ramas y tags grandes.

## Auditoria de seguridad

El historial de Git puede contener accidentalmente informacion sensible: contraseñas, claves API, tokens, datos personales. Una vez comprometido, el dato queda en el historial incluso si se elimina en un commit posterior.

**Detectar credenciales en el historial:**

```bash
# Buscar tokens o claves en commits anteriores
git log -p -S "API_KEY" --all
git log -p -S "password" --all
git log -p -S "secret" --all

# Buscar archivos que contengan secrets
git log --all --full-history -- "*.env"
git log --all --full-history -- "*.pem"
```

**Herramientas de escaneo:**

- **GitLeaks:** escanea el historial en busca de secretos.
- **TruffleHog:** detecta tokens y credenciales.
- **GitHub Secret Scanning:** escaneo automatico en repositorios.
- **GitGuardian:** monitoreo continuo de secretos.

```bash
# Escaneo con GitLeaks
gitleaks detect --source . --verbose

# Escaneo con TruffleHog
trufflehog git file://. --only-verified
```

**Si se detecta una credencial comprometida:**

1. Revocar la credencial inmediatamente en el servicio correspondiente.
2. Generar una nueva credencial.
3. Eliminar la credencial del historial con `git filter-repo` o BFG Repo-Cleaner.
4. Hacer force push para sobrescribir el historial.
5. Notificar al equipo.

```bash
# Eliminar un archivo del historial completo con BFG
bfg --delete-files "*.env"
bfg --replace-text passwords.txt
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push --force --all
```

> La recuperacion en Git es mas facil que en cualquier otro VCS: casi todo es reversible. Pero la seguridad es mas dificil: el historial inmutable que hace Git tan potente tambien significa que los secretos subidos por error permanecen para siempre a menos que se reescriba el historial. Escanear periodicamente el historial es una practica de seguridad obligatoria.

