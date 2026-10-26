# Tipos de sistemas de archivos

Los sistemas de archivos se clasifican según su diseño, estructura interna y casos de uso. Cada uno está optimizado para diferentes patrones de acceso, tamaños de archivo y requisitos de integridad. La elección del sistema de archivos adecuado depende del tipo de datos, el volumen de almacenamiento, los requisitos de rendimiento y las necesidades de recuperación ante fallos.

Linux soporta una amplia variedad de sistemas de archivos, tanto nativos como de otros sistemas operativos, lo que permite montar y acceder a volúmenes de casi cualquier plataforma.

## ext4

ext4 es el sistema de archivos nativo de Linux y el más utilizado en distribuciones generales. Es una evolución de ext3, que a su vez evolucionó de ext2. Características principales:

- Journaling completo para integridad de metadatos y datos.
- Soporte para volúmenes de hasta 1 EB y archivos de hasta 16 TB.
- Extents (rangos de bloques contiguos) en lugar de punteros indirectos, mejorando el rendimiento para archivos grandes.
- Allocación retardada (*delayed allocation*) para reducir la fragmentación.
- Soporte para snapshots con herramientas como LVM.
- Límite de 32.000 subdirectorios por directorio (ampliable con directorios hash).

ext4 es una opción sólida para la mayoría de los casos de uso: discos de sistema, particiones de usuario, servidores generales. Su madurez y fiabilidad lo convierten en la opción predeterminada segura.

## XFS

XFS es un sistema de archivos de alto rendimiento diseñado originalmente por SGI para IRIX y posteriormente portado a Linux. Está optimizado para archivos grandes y operaciones de E/S paralelas:

- Journaling con separación de metadatos y datos.
- Soporte para volúmenes de hasta 8 EB y archivos de 8 EB.
- Asignación dinámica de inodos (*dynamic inode allocation*).
- Almacenamiento en línea (*online defragmentation*).
- Soporte nativo para cuotas de disco.
- Excelente rendimiento con archivos grandes y operaciones de E/S concurrentes.

XFS es ideal para servidores de archivos multimedia, bases de datos grandes y sistemas con alta concurrencia de E/S. Su debilidad es la gestión de archivos pequeños muy numerosos, donde ext4 puede ser más eficiente.

## Btrfs

Btrfs (*B-tree file system*) es un sistema de archivos moderno con características avanzadas de gestión de volumen e integridad:

- Copy-on-write (CoW): nunca modifica bloques existentes en su ubicación.
- Snapshots instantáneos y eficientes.
- Compressión transparente (zlib, lzo).
- Verificación de integridad con checksums para datos y metadatos.
- RAID por software integrado (RAID 0, 1, 10, 5, 6).
- Clone de archivos y directorios.
- Subvolúmenes para organizar el almacenamiento.

Btrfs está siendo adoptado como sistema de archivos predeterminado en algunas distribuciones (Fedora Workstation desde 2020). Es ideal para sistemas que requieren snapshots, compresión o verificación de integridad.

## ZFS

ZFS es un sistema de archivos y gestor de volumen combinado, originalmente desarrollado por Sun Microsystems. Aunque no es nativo de Linux, se puede usar a través de la capa de compatibilidad OpenZFS:

- Copy-on-write con integridad de datos garantizada.
- RAID-Z (variante de RAID por software) con protección contra corrupción silenciosa.
- Compresión LZ4 y ZSTD.
- Deduplicación de datos.
- Snapshots y clones.
- Cuotas de disco y contenedores de datos.
- Soporte para volúmenes de hasta 256 ZiB.

ZFS es la opción preferida para servidores NAS, sistemas de almacenamiento empresarial y cualquier entorno donde la integridad de datos es crítica. Su principal desventaja es el consumo de memoria (requiere al menos 8 GB recomendados) y la complejidad de su configuración.

## NTFS y FAT32

**NTFS** es el sistema de archivos nativo de Windows. Linux puede leer y escribir en NTFS a través del driver `ntfs3` (nativo del kernel) o NTFS-3G (en espacio de usuario). Es necesario para acceder a particiones de Windows en configuraciones de doble arranque o discos externos formateados en Windows.

**FAT32** es un sistema de archivos antiguo y ampliamente compatible, utilizado en dispositivos extraíbles (USB, tarjetas SD). Su principal limitación es el tamaño máximo de archivo de 4 GB y la falta de journaling, lo que lo hace vulnerable a la corrupción ante fallos de energía. **exFAT** es la evolución de FAT32 que elimina la limitación de 4 GB y es el estándar para tarjetas SD de gran capacidad.

> La elección del sistema de archivos es una decisión de diseño con implicaciones a largo plazo. Cambiar el sistema de archivos de un volumen en producción requiere formateo y migración de datos, un proceso costoso y arriesgado. Es recomendable evaluar los requisitos antes de la implementación.

