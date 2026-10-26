# Operaciones de sincronizacion

Las operaciones de sincronizacion permiten mover cambios entre el repositorio local y los repositorios remotos. Clonar, obtener (*fetch*), extraer (*pull*) y enviar (*push*) son las cuatro operaciones fundamentales que conectan tu trabajo local con el resto del equipo.

Estas operaciones son el puente entre el trabajo individual y la协作 colectiva: cada una tiene un proposito especifico y un momento adecuado para su uso.

## Clonar (clone)

`git clone` descarga una copia completa de un repositorio remoto, incluyendo todo el historial, ramas y etiquetas. Es la primera operacion que realizas cuando quieres contribuir a un proyecto existente.

```bash
# Clonar por HTTPS
git clone https://github.com/usuario/proyecto.git

# Clonar por SSH
git clone git@github.com:usuario/proyecto.git

# Clonar en un directorio especifico
git clone https://github.com/usuario/proyecto.git mi-carpeta

# Clonar solo una rama
git clone -b develop https://github.com/usuario/proyecto.git
```

El comando `clone` ejecuta implicitamente `git init`, `git remote add origin <url>` y `git fetch origin`, seguido de `git checkout main`.

## Fetch

`git fetch` descarga los cambios del repositorio remoto sin fusionarlos con tu trabajo local. Es la operacion mas segura de las tres, ya que no modifica tu directorio de trabajo ni tu area de preparacion.

```bash
# Obtener todos los cambios de todos los remotos
git fetch --all

# Obtener solo de origin
git fetch origin

# Obtener y limpiar ramas remotas eliminadas
git fetch --prune
```

Despues de `fetch`, puedes revisar los cambios antes de fusionarlos:

```bash
# Ver los commits que trajiste
git log main..origin/main

# Ver los archivos modificados
git diff main origin/main
```

## Pull

`git pull` es la combinacion de `fetch` + `merge`. Obtiene los cambios del remoto y los fusiona automaticamente con tu rama actual. Es equivalente a:

```bash
git fetch origin
git merge origin/main
```

```bash
# Pull basico
git pull origin main

# Pull con rebase (en lugar de merge)
git pull --rebase origin main

# Configurar pull con rebase por defecto
git config --global pull.rebase true
```

**fetch vs pull:**

| Aspecto | fetch | pull |
|---|---|---|
| Obtiene cambios | Si | Si |
| Fusiona automaticamente | No | Si |
| Modifica directorio de trabajo | No | Si |
| Seguro | Siempre | Puede causar conflictos |
| Uso recomendado | Revision previa | Confianza en el remoto |

## Push

`git push` envia tus commits locales al repositorio remoto. Solo envia commits que existen en tu repositorio local pero no en el remoto.

```bash
# Enviar rama actual al remoto
git push origin main

# Enviar y configurar rama de seguimiento
git push -u origin feature/nueva

# Enviar etiquetas
git push origin --tags
```

**El error mas comun:** si el remoto tiene commits que tu local no tiene, Git rechazara el push para no sobrescribir el historial. Solucion: hacer `pull` antes de `push`.

```bash
# Error tipico
! [rejected]        main -> main (non-fast-forward)
error: failed to push some refs to 'origin'
hint: Updates were rejected because the tip of your current branch is behind

# Solucion
git pull origin main
# Resolver conflictos si los hay
git push origin main
```

## Flujo basico de sincronizacion

```bash
# 1. Obtener cambios del remoto
git fetch origin

# 2. Revisar los cambios
git log main..origin/main

# 3. Fusionar o rebase
git merge origin/main
# O: git rebase origin/main

# 4. Trabajar localmente
git add .
git commit -m "Mi cambio"

# 5. Enviar al remoto
git push origin main
```

> El flujo correcto de sincronizacion es fetch antes de pull, y pull antes de push. Esta secuencia garantiza que trabajas con los cambios mas recientes y que tus envios no sobrescriben el trabajo de otros.

