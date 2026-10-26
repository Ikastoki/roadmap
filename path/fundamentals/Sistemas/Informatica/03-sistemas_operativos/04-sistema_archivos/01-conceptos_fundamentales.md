# Conceptos fundamentales del sistema de archivos

El sistema de archivos es la capa del sistema operativo responsable de organizar, almacenar, recuperar y gestionar datos en dispositivos de almacenamiento. Proporciona una abstracción que oculta los detalles del hardware de almacenamiento (sectores, bloques, direcciones LBA) y presenta al usuario y a las aplicaciones una interfaz simple de archivos y directorios.

El diseño del sistema de archivos tiene un impacto directo en la fiabilidad, rendimiento y capacidad de recuperación del sistema. Un buen sistema de archivos garantiza la integridad de los datos incluso ante fallos de hardware, proporciona mecanismos eficientes de acceso y permite la gestión eficaz de grandes volúmenes de datos.

## Qué es un sistema de archivos

Un sistema de archivos es tanto la estructura de datos que organiza los datos en el disco como el conjunto de mecanismos del kernel que gestionan esa estructura. Incluye:

- **Mecanismos de almacenamiento:** cómo se distribuyen los bloques de datos en el disco y cómo se representan las relaciones jerárquicas (directorios).
- **Estructura de metadatos:** información sobre archivos (tamaño, permisos, fechas de modificación, propietario) que se almacena en inodos, entradas de directorio o estructuras similares.
- **Mecanismos de integridad:** journaling, copy-on-write o checksums que protegen contra corrupción de datos.
- **Interfaz de acceso:** las llamadas al sistema que las aplicaciones utilizan para crear, leer, escribir y eliminar archivos.

## Inodos

En los sistemas de archivos tipo Unix (ext4, XFS, Btrfs), la unidad básica de metadatos es el **inodo** (*index node*). Cada archivo o directorio tiene un inodo que contiene:

- Tipo de archivo (regular, directorio, enlace simbólico, etc.).
- Permisos de acceso (lectura, escritura, ejecución para propietario, grupo y otros).
- Propietario y grupo.
- Tamaño del archivo.
- Timestamps (creación, último acceso, última modificación, último cambio de metadatos).
- Punteros a los bloques de datos que contienen el contenido del archivo.

El nombre del archivo no está en el inodo; está en la entrada de directorio que apunta al inodo. Esta separación permite que múltiples nombres (enlaces duros) apunten al mismo inodo.

## Journaling

El *journaling* es un mecanismo que registra los cambios pendientes en un área especial del disco (el *journal*) antes de aplicarlos a la estructura principal del sistema de archivos. Si el sistema falla durante una operación de escritura, el journal permite recuperarse rápidamente revirtiendo o completando las operaciones pendientes.

El flujo típico de una operación con journaling es:

1. Se escribe la operación en el journal (commit).
2. Se confirma que la operación está registrada en el journal (*journal commit*).
3. Se aplican los cambios a la estructura principal del sistema de archivos.
4. Se marca la operación como completada en el journal.

Si ocurre un fallo entre los pasos 2 y 3, el sistema de archivos puede revisar el journal y reintentar o revertir las operaciones incompletas, garantizando la integridad de la estructura. Sin journaling, un fallo puede dejar el sistema de archivos en un estado inconsistente que requiere una verificación completa con `fsck`, un proceso que puede tardar horas en discos grandes.

## Journaling vs copy-on-write

El journaling tradicional escribe los cambios dos veces: una en el journal y otra en la estructura principal. El *copy-on-write* (CoW) adopta un enfoque diferente: nunca modifica los bloques existentes en su ubicación original, sino que escribe los nuevos bloques en ubicaciones libres y actualiza los punteros de la raíz al final. Si falla durante la operación, los bloques antiguos permanecen intactos.

Btrfs y ZFS utilizan copy-on-write, lo que proporciona integridad de datos, snapshots eficientes y recuperación automática de errores. El coste es una mayor complejidad y un rendimiento de escritura potencialmente menor en ciertos patrones de acceso.

> El sistema de archivos es una de las capas más críticas de un sistema operativo: un fallo en su diseño o implementación puede causar pérdida de datos catastrófica. La elección del sistema de archivos (ext4, XFS, Btrfs, ZFS) debe considerar el caso de uso: ext4 para generalidad, XFS para archivos grandes, Btrfs para snapshots y gestión de volumen, ZFS para integridad de datos y almacenamiento escalable.

