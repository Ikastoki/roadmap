<h1 align="center">Conflictos</h1>

<h2>📑 Contenido</h2>

- [Conflictos](#conflictos)
- [Conflictos más comunes](#conflictos-más-comunes)
  - [Conflicto de fusión(Merge Conflict)](#conflicto-de-fusiónmerge-conflict)
  - [Error de resolución de conflicto](#error-de-resolución-de-conflicto)
  - [Commit vacío(Empty Commit)](#commit-vacíoempty-commit)
  - [Olvidar agregar archivos(Forget to Add Files)](#olvidar-agregar-archivosforget-to-add-files)
  - [Ramas no sincronizadas(Branches Not Up-to-Date)](#ramas-no-sincronizadasbranches-not-up-to-date)
  - [Referencia no válida(Invalid Reference)](#referencia-no-válidainvalid-reference)
  - [Cambios no confirmados(Uncommitted Changes)](#cambios-no-confirmadosuncommitted-changes)
  - [Historial de confirmaciones confuso(Messy Commit History)](#historial-de-confirmaciones-confusomessy-commit-history)
  - [Repositorio corrupto(Corrupted Repository)](#repositorio-corruptocorrupted-repository)
  - [Conflictos de configuración(Configuration Conflicts)](#conflictos-de-configuraciónconfiguration-conflicts)

## Conflictos

Los conflictos en Git y GitHub generalmente ocurren cuando hay cambios conflictivos en el código entre diferentes ramas o entre el repositorio local y el remoto.

## Conflictos más comunes

### Conflicto de fusión(Merge Conflict)

Esto ocurre cuando Git no puede fusionar automáticamente dos ramas debido a cambios conflictivos en el mismo archivo o línea. Para resolverlo, debes abrir el archivo en conflicto, resolver manualmente los cambios conflictivos y luego realizar un commit para finalizar la fusión.

### Error de resolución de conflicto

A veces, al resolver conflictos, puedes cometer errores, como dejar marcadores de conflicto sin resolver o eliminar el código necesario. Debes asegurarte de revisar cuidadosamente tus cambios antes de confirmar la resolución.

### Commit vacío(Empty Commit)

Esto sucede cuando realizas un commit sin cambios. Puede ocurrir accidentalmente al ejecutar git commit sin agregar ningún archivo previamente con git add. Evita estos commits vacíos y asegúrate de que tus cambios estén listos antes de confirmarlos.

### Olvidar agregar archivos(Forget to Add Files)

Puedes cometer el error de confirmar sin agregar archivos nuevos o modificados previamente con git add. Asegúrate de agregar los archivos necesarios antes de confirmar los cambios.

### Ramas no sincronizadas(Branches Not Up-to-Date)

Puedes encontrarte en situaciones en las que tu rama local no esté actualizada con la rama remota. Debes usar git pull para obtener los cambios más recientes de la rama remota antes de continuar con tu trabajo.

### Referencia no válida(Invalid Reference)

Esto puede ocurrir si intentas acceder a una rama, commit o etiqueta que no existe. Asegúrate de que la referencia que estás utilizando sea correcta.

### Cambios no confirmados(Uncommitted Changes)

Intentar cambiar de rama o realizar otras operaciones cuando tienes cambios sin confirmar puede generar conflictos o problemas. Confirma tus cambios o guárdalos en una rama temporal antes de cambiar de contexto.

### Historial de confirmaciones confuso(Messy Commit History)

Si no sigues buenas prácticas al realizar commits, como proporcionar mensajes claros y descriptivos, tu historial de confirmaciones puede volverse difícil de entender. Intenta mantener tu historial limpio y bien documentado.

### Repositorio corrupto(Corrupted Repository)

Esto es poco común, pero puede ocurrir si los archivos internos de Git se dañan o corrompen debido a fallas en el sistema o errores graves. En casos extremos, es posible que necesites clonar un nuevo repositorio y migrar tus cambios manualmente.

### Conflictos de configuración(Configuration Conflicts)

Pueden surgir conflictos de configuración si tienes configuraciones locales y globales que entran en conflicto. Resuelve estos conflictos revisando y ajustando tus configuraciones.


