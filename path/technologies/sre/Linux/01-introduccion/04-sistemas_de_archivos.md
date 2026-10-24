# Sistemas de archivos en Linux

En Linux, todo es un archivo: los documentos, los directorios, los dispositivos de bloque, los sockets de red, incluso los procesos expuestos en `/proc`. Esta abstraccion unifica la interfaz de entrada/salida y simplifica la interaccion con el sistema. Pero detras de esa uniformidad se esconde una capa de complejidad notable: el sistema de archivos propiamente dicho, el que decide como se almacenan, organizan y recuperan los datos en el disco.

## Sistemas de archivos nativos de Linux

### ext4

ext4 (fourth extended filesystem) es el sistema de archivos por defecto en la mayoria de las distribuciones Linux. Es la evolucion de ext3, que a su vez provenia de ext2, y representa la madurez de una linea de desarrollo que arranca en 1992. ext4 soporta archivos de hasta 16 TB y volumenes de hasta 1 EB, aunque los limites practicos dependen del tamano de bloque.

Las caracteristicas que distinguen a ext4 incluyen:
- **Extents**: reemplazan el esquema de bloques individuales de ext3. Un extent es un rango contiguo de bloques, lo que reduce la fragmentacion y acelera las operaciones con archivos grandes.
- **Asignacion diferida (delayed allocation)**: el sistema retrasa la escritura en disco hasta que realmente es necesaria, agrupando los datos y mejorando el rendimiento.
- **Compatibilidad hacia atras**: ext4 puede montar sistemas de archivos ext2 y ext3, aunque pierde algunas ventajas de rendimiento.
- **Listas de acceso (FACL) y atributos extendidos (xattr)**: soporte completo para permisos avanzados y metadatos arbitrarios.

```bash
# Crear un sistema ext4
mkfs.ext4 /dev/sdb1

# Ver informacion del sistema de archivos
tune2fs -l /dev/sda1 | grep -E "Filesystem|Block size|Inode size"

# Comprobar y reparar (el sistema debe estar desmontado)
fsck.ext4 /dev/sdb1
```

### XFS

XFS fue desarrollado por Silicon Graphics (SGI) para su sistema operativo IRIX y portado a Linux en 2001. Esta disenado para escalar a volumenes y archivos de gran tamano: soporta archivos de hasta 8 EB y sistemas de archivos de 8 EB en teoria. Es la opcion preferida para servidores de archivos, bases de datos y entornos con archivos de gran tamano.

XFS utiliza un arbol B+ (B-tree) como estructura de datos central para gestionar las extents, los inodos y el espacio libre. Esto proporciona un rendimiento consistente incluso cuando el sistema de archivos esta muy fragmentado o lleno. La asignacion de espacio es altamente paralelizable gracias a las AGs (Allocation Groups), subvolumenes independientes dentro del mismo sistema de archivos.

A diferencia de ext4, XFS no permite reducir el tamano de un volumen (shrink) sin desmontarlo y usar herramientas externas. Esta limitacion es importante al dimensionar particiones.

```bash
# Crear un sistema XFS
mkfs.xfs /dev/sdc1

# Aumentar el tamano (XFS solo soporta grow)
xfs_growfs /mount/point

# Ver informacion
xfs_info /mount/point
```

### Btrfs

Btrfs (B-tree File System, pronunciado "Butter FS") adopta un enfoque radicalmente distinto: es un sistema de archivos copy-on-write (CoW) que integra funcionalidades que en otros sistemas requieren capas adicionales: snapshots, compresion transparente, sumas de verificacion en datos y metadatos, subvolumenes, y soporte nativo para RAID 0, 1, 5, 6 y 10.

El copy-on-write significa que cuando se modifica un archivo, los datos originales no se sobrescriben: se escriben en una nueva ubicacion y el puntero se actualiza atomicamente. Esto permite snapshots instantaneos y consistentes, ya que el snapshot solo conserva los bloques originales que han sido reemplazados.

```bash
# Crear un sistema Btrfs
mkfs.btrfs /dev/sdd1

# Crear un subvolumen y un snapshot
btrfs subvolume create /mnt/datos
btrfs subvolume snapshot /mnt/datos /mnt/datos-snapshot-20260101

# Verificar sumas de comprobacion
btrfs scrub start /mnt/datos
```

> Btrfs es el sistema de archivos por defecto en openSUSE y Fedora Workstation, y ha sido adoptado por Synology y otros fabricantes de NAS. Aunque ha madurado significativamente, ext4 sigue siendo la opcion conservadora para sistemas donde la fiabilidad es critica.

## Montaje de sistemas de archivos

Montar un sistema de archivos es el proceso de hacer accesible su contenido en un directorio del arbol de directorios. El comando `mount` asocia un dispositivo (o un UUID) a un punto de montaje.

```bash
# Montaje basico
mount /dev/sdb1 /mnt/disco

# Montaje con opciones
mount -t ext4 -o noatime,nodiratime /dev/sdb1 /mnt/disco

# Desmontar
umount /mnt/disco
```

El archivo `/etc/fstab` define los sistemas de archivos que se montan automaticamente al arrancar. Cada linea tiene seis campos:

```
# <dispositivo>  <punto_montaje>  <tipo>  <opciones>  <dump>  <pass>
UUID=abc123     /                ext4    errors=remount-ro  0       1
UUID=def456     /home            ext4    defaults           0       2
/dev/sdb1       /mnt/data        xfs     defaults           0       0
192.168.1.10:/export /mnt/nfs    nfs     defaults           0       0
```

El campo `pass` indica el orden de comprobacion del sistema de archivos en el arranque: 1 para la raiz, 2 para los demas, 0 para no comprobar.

### UUID y LABEL

Los identificadores de dispositivos por nombre (/dev/sda1, /dev/sdb1) no son fiables porque el orden de deteccion del kernel puede variar entre arranques. La solucion es usar UUID (Universally Unique Identifier) o LABEL para referenciar los sistemas de archivos.

```bash
# Obtener UUID y LABEL de los dispositivos
blkid

# Resultado tipico:
# /dev/sda1: UUID="a1b2c3d4-..." TYPE="ext4" PARTLABEL="root"
# /dev/sdb1: LABEL="DATA" UUID="e5f6g7h8-..." TYPE="xfs"
```

En `/etc/fstab` se prefiere UUID sobre nombres de dispositivo. Las particiones GPT tambien tienen PARTUUID, que identifica la particion independientemente del sistema de archivos.

## LVM (Logical Volume Manager)

LVM abstrae el almacenamiento fisico en tres capas: los discos o particiones se convierten en volumes fisicos (PV), que se agrupan en grupos de volumenes (VG), de los cuales se extraen volumenes logicos (LV) que el sistema usa como dispositivos de bloque.

La utilidad practica de LVM es enorme: permite redimensionar volumenes logicos sin particionar discos, agregar nuevos discos a un grupo existente, crear snapshots a nivel de bloque, y mover datos entre discos sin interrumpir el servicio.

```bash
# Crear un volumen fisico
pvcreate /dev/sdb /dev/sdc

# Crear un grupo de volumenes
vgcreate vg_datos /dev/sdb /dev/sdc

# Crear un volumen logico
lvcreate -L 100G -n lv_home vg_datos

# Redimensionar (online)
lvextend -L +50G /dev/vg_datos/lv_home
resize2fs /dev/vg_datos/lv_home   # ext4
xfs_growfs /home                  # XFS

# Ver estructura
pvdisplay
vgdisplay
lvdisplay
```

Las extensiones (PE, Physical Extents) son los bloques minimos que LVM gestiona, tipicamente de 4 MB. Cada volumen logico esta compuesto por uno o varios PE que pueden residir en discos fisicos distintos.

> LVM no sustituye al RAID. Ambos son complementarios: LVM proporciona flexibilidad en la gestion del espacio, mientras que RAID proporciona tolerancia a fallos. Lo habitual es crear primero los arrays RAID y despues construir LVM sobre ellos.

## RAID por software

RAID (Redundant Array of Independent Disks) combina multiples discos en una sola unidad logica para mejorar la redundancia, el rendimiento o ambos. Linux implementa RAID por software a traves del subsistema md (multiple device).

Los niveles mas utilizados:
- **RAID 0 (striping)**: los datos se distribuyen entre los discos. No hay redundancia, pero el rendimiento de lectura/escritura se multiplica por el numero de discos.
- **RAID 1 (mirroring)**: cada disco contiene una copia identica de los datos. Tolerancia a fallo de N-1 discos, rendimiento de lectura mejorado.
- **RAID 5**: striping con paridad distribuida. Tolerancia a un fallo con al menos 3 discos. Rendimiento de lectura bueno, escritura penalizada por el calculo de paridad.
- **RAID 6**: similar a RAID 5 pero con doble paridad. Tolerancia a dos fallos simultaneos.
- **RAID 10**: combinacion de RAID 1 y RAID 0. Requiere al menos 4 discos. Tolerancia a fallos y rendimiento.

```bash
# Crear un RAID 1 con dos discos
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Ver estado del array
cat /proc/mdstat
mdadm --detail /dev/md0

# Anadir un disco de respaldo (hot spare)
mdadm --add /dev/md0 /dev/sdd

# Guardar configuracion
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```

## Snapshots

Un snapshot es una copia instantanea del estado de un sistema de archivos en un momento dado. Los snapshots permiten realizar backups consistentes, restaurar archivos eliminados accidentalmente o probar cambios con la garantia de poder revertirlos.

En LVM, los snapshots funcionan mediante copy-on-write. Cuando se crea un snapshot, LVM marca todos los bloques del volumen original como "compartidos". Al modificarse un bloque, LVM copia el bloque original al area del snapshot antes de escribir el nuevo. El snapshot solo ocupa espacio proporcional a los cambios realizados.

```bash
# Crear un snapshot LVM (10 GB de area CoW)
lvcreate -L 10G -s -n snap_home /dev/vg_datos/lv_home

# Montar el snapshot (read-only para backup)
mount -o ro /dev/vg_datos/snap_home /mnt/backup

# Eliminar el snapshot
lvremove /dev/vg_datos/snap_home
```

Btrfs ofrece snapshots nativos a nivel de subvolumen, que son instantaneos y casi sin coste de espacio inicial:

```bash
# Snapshot de subvolumen Btrfs
btrfs subvolume snapshot -r /mnt/datos /mnt/datos/snapshots/antes-de-cambios
```

## Cuotas de disco

Las cuotas de disco limitan el espacio que un usuario o grupo puede ocupar en un sistema de archivos. Son esenciales en entornos multiusuario donde un usuario podria saturar el disco compartido.

Linux soporta dos niveles de limitacion: cuota blanda (soft) y cuota dura (hard). El usuario recibe advertencias al superar la blanda y tiene un periodo de gracia antes de que se le impida escribir mas. La cuota dura es un limite absoluto.

```bash
# Instalar herramientas de cuotas (Debian/Ubuntu)
apt install quota

# Habilitar cuotas en /etc/fstab
# /dev/sdb1 /home ext4 defaults,usrquota,grpquota 0 0

# Remontar y generar bases de datos de cuotas
mount -o remount /home
quotacheck -cum /home
quotaon /home

# Asignar cuota a un usuario
edquota -u usuario

# Ver cuotas en uso
quota -s usuario
repquota -a
```

```bash
# Ejemplo de configuracion con edquota
# Disk quotas for user usuario (uid 1001):
# /dev/sdb1: blocks in use: 1024, limits (soft = 5000000, hard = 6000000)
#            inodes in use: 150, limits (soft = 10000, hard = 12000)
```

> Las cuotas funcionan a nivel de sistema de archivos. Si un usuario puede escribir en varios sistemas de archivos, cada uno tiene su propia cuota independiente. project quotas extienden el concepto para limitar el espacio usado por un conjunto de directorios, independientemente del usuario propietario.

