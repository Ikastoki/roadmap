# Arquitectura conceptual de Git

Git organiza el trabajo en tres areas principales: el directorio de trabajo, el area de preparacion y el repositorio local. Comprender como interactuan estas tres areas es fundamental para usar Git de forma efectiva, ya que cada operacion de Git mueve datos entre estas areas de forma especifica.

Este modelo de tres areas es lo que hace a Git un sistema flexible y potente: permite preparar cambios de forma selectiva, crear instantaneas atomicas y mantener un historial completo del proyecto.

## Directorio de trabajo (Working Directory)

El directorio de trabajo es tu copia local y activa de los archivos del proyecto. Es donde editas, creas y eliminas archivos. Estos archivos son los que el sistema de archivos muestra normalmente; Git los trata como una representacion del proyecto en un momento dado.

Cuando clonas un repositorio, el directorio de trabajo contiene la version mas reciente de todos los archivos (la rama activa). Los cambios que realices en estos archivos son locales y no se reflejan en el historial de Git hasta que los confirmes (*commit*).

## Area de preparacion (Staging Area / Index)

El area de preparacion es una zona intermedia donde se seleccionan los cambios que formaran parte de la proxima instantanea. No es un directorio fisico; es una estructura de datos interna de Git que almacena un snapshot de los archivos que se incluiran en el proximo commit.

El area de preparacion permite crear commits atomicos: puedes modificar 10 archivos, preparar 3 de ellos y confirmar solo esos 3, dejando los demas cambios sin confirmar. Esta granularidad es valiosa para crear historiales limpios y organizados.

## Repositorio local (Repository)

El repositorio local es la base de datos de Git que almacena todas las instantaneas (commits) del proyecto, incluyendo todo el historial. Se encuentra en el directorio `.git/` en la raiz del proyecto y contiene todos los objetos Git: commits, arboles, blobs y referencias.

El repositorio local es completamente funcional sin conexion a internet. Puedes crear branches, hacer commits, ver el historial y fusonar ramas todo offline. Solo necesitas conexion cuando quieras sincronizar con un repositorio remoto.

## Ciclo de vida del archivo

Los archivos en un repositorio Git pasan por cuatro estados:

**Untracked (no rastreado):** el archivo existe en el directorio de trabajo pero Git no lo conoce. No esta en el area de preparacion ni en el repositorio.

**Unmodified (sin cambios):** el archivo esta rastreado por Git y su contenido es identico al ultimo commit.

**Modified (modificado):** el archivo esta rastreado pero su contenido ha cambiado desde el ultimo commit. Los cambios estan solo en el directorio de trabajo.

**Staged (preparado):** el archivo ha sido marcado para incluirse en el proximo commit. Los cambios estan en el area de preparacion.

```
Directorio de trabajo          Area de preparacion        Repositorio
┌──────────────┐              ┌──────────────┐          ┌──────────────┐
│  archivos    │   git add    │  archivos    │  git     │  instantanea │
│  modificados │ ──────────→ │  preparados  │ commit   │  (commit)    │
│              │              │              │ ────────→│              │
└──────────────┘              └──────────────┘          └──────────────┘
```

## Modelo de almacenamiento: snapshots vs deltas

A diferencia de sistemas como SVN que almacenan deltas (diferencias entre versiones consecutivas), Git almacena **instantaneas** (*snapshots*). Cada commit contiene una referencia completa del estado de todos los archivos del proyecto en ese momento.

Git es eficiente porque no duplica archivos que no han cambiado: si un archivo es identico al del commit anterior, almacena una referencia al objeto existente en lugar de una copia nueva. Solo los archivos que han cambiado ocupan espacio adicional.

```bash
# Ver el contenido de un commit
git show HEAD

# Ver los archivos modificados en un commit
git diff-tree --no-commit-id -r abc1234

# Ver el arbol de archivos de un commit
git ls-tree -r HEAD --name-only
```

> Comprender las tres areas de Git (directorio de trabajo, area de preparacion, repositorio) es la clave para usar Git de forma consciente. Cada operacion de Git mueve datos entre estas areas: `git add` mueve del directorio al area de preparacion; `git commit` mueve del area de preparacion al repositorio; `git checkout` mueve del repositorio al directorio de trabajo.

