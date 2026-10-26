# Manejo de ramas remotas

Las ramas remotas son referencias a las ramas del repositorio remoto que Git almacena localmente. Te permiten ver que ramas existen en el remoto, que cambios tienen y como se comparan con tu trabajo local. Gestionar correctamente las ramas remotas es esencial para mantener un repositorio limpio y organizado.

## Ramas de seguimiento (tracking branches)

Cuando clonas un repositorio o haces checkout de una rama remota, Git crea una rama local que "sigue" (*tracks*) a la rama remota. Esta relacion permite que `git pull` y `git push` sepan automaticamente que rama remota utilizar.

```bash
# Ver ramas locales y su rastreo
git branch -vv
# * main       a1b2c3d [origin/main] Ultimo commit
#   feature    e4f5g6h [origin/feature: ahead 2] Mis commits
```

**Crear rama de seguimiento:**

```bash
# Clonar y configurar seguimiento automaticamente
git checkout -b feature origin/feature

# O configurar seguimiento para una rama existente
git branch --set-upstream-to=origin/main main
```

## Ver ramas remotas

```bash
# Listar ramas remotas
git branch -r
# origin/main
# origin/develop
# origin/feature/login

# Listar todas (locales y remotas)
git branch -a

# Ver ultima actividad de cada rama remota
git branch -r --merged main     # Ramas remotas ya fusionadas con main
git branch -r --no-merged main  # Ramas remotas con cambios pendientes
```

## Limpiar ramas remotas eliminadas

Cuando un colaborador elimina una rama remota, tu repositorio local sigue conservando la referencia. `git fetch --prune` elimina las referencias a ramas que ya no existen en el remoto.

```bash
# Eliminar referencias a ramas remotas eliminadas
git fetch --prune

# Ver que ramas se eliminarian (dry-run)
git fetch --prune --dry-run
```

## Sincronizar ramas con el remoto

```bash
# Enviar una nueva rama al remoto
git push -u origin feature/nueva

# Eliminar rama remota
git push origin --delete feature/terminada

# Renombrar rama remota
git branch -m old-name new-name
git push origin old-name:new-name
git push origin --delete old-name
```

## Resolucion de conflictos entre local y remoto

Cuando tu historial local y el remoto han divergido (ambos tienen commits que el otro no tiene), Git requiere una fusion o rebase antes de permitir el push.

```bash
# Obtener cambios
git fetch origin

# Ver la divergencia
git log --oneline HEAD..origin/main
git log --oneline origin/main..HEAD

# Fusionar (crea commit de fusion)
git merge origin/main

# O rebase (reescribe historial local para que sea lineal)
git rebase origin/main
```

> Las ramas remotas son la memoria compartida del equipo. Mantenerlas limpias (eliminar ramas fusionadas, hacer fetch --prune periodicamente) evita confusion y facilita la navegacion del historial. Un repositorio con decenas de ramas obsoletas es dificil de entender y mantener.

