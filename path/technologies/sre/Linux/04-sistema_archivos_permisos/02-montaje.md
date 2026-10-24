# Montaje de sistemas de archivos

En Linux, acceder al contenido de un dispositivo de almacenamiento requiere un paso previo: el montaje. Este proceso asocia un dispositivo (una particion de disco, un pendrive, una imagen ISO) a un directorio del arbol del sistema, haciendo que su contenido sea accesible desde ese punto. Sin montaje, el dispositivo existe como nodo en `/dev` pero no como estructura de archivos navegable. Comprender el montaje es esencial para la gestion de almacenamiento, la administracion de discos y la resolucion de problemas de espacio.

## El comando mount

`mount` es la herramienta principal para montar sistemas de archivos. Su sintaxis basica es:

```bash
mount -t tipo_dispositivo punto_montaje
mount /dev/sdb1 /mnt/disco
```

Opciones habituales:
- `-t fstype`: especifica el tipo de sistema de archivos (ext4, xfs, ntfs, vfat). Si se omite, mount intenta detectarlo automaticamente.
- `-o opciones`: permite pasar opciones de montaje separadas por comas: `ro` (solo lectura), `rw` (lectura/escritura), `noexec` (no permitir ejecucion de binarios), `uid=1000`, `gid=1000`.

```bash
mount -t ext4 -o ro,noexec /dev/sdb1 /mnt/disco
```

Sin argumentos, `mount` muestra todos los sistemas de archivos montados actualmente:

```bash
mount
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
/dev/sda2 on / type ext4 (rw,relatime,errors=remount-ro)
/dev/sda1 on /boot/efi type vfat (rw,relatime,fmask=0077,dmask=0077)
```

Cada linea muestra: dispositivo, punto de montaje, tipo de sistema de archivos y opciones de montaje entre parentesis.

## El comando umount

Para desmontar un sistema de archivos se usa `umount` (sin la 'n' de unmount). El sistema de archivos no debe estar en uso para poder desmontarlo.

```bash
umount /mnt/disco
umount /dev/sdb1
```

Si el dispositivo esta ocupado por algun proceso, el comando falla con el mensaje "target is busy". Para identificar que proceso lo esta usando:

```bash
lsof /mnt/disco
fuser -v /mnt/disco
```

La opcion `-l` (lazy) fuerza el desmontaje cuando el dispositivo deje de estar ocupado:

```bash
umount -l /mnt/disco
```

> Forzar un desmontaje con `-l` o `-f` puede provocar perdida de datos si hay escrituras pendientes. Solo debe usarse cuando el desmontaje normal falla y se tiene certeza de que no hay operaciones criticas en curso.

## El archivo /etc/fstab

El archivo `/etc/fstab` (filesystem table) define los sistemas de archivos que se montan automaticamente durante el arranque. Cada linea describe un dispositivo, su punto de montaje, tipo, opciones y parametros de volcado y verificacion.

```bash
# <dispositivo>    <punto_montaje>  <tipo>  <opciones>  <dump>  <pass>
UUID=abc123-...   /                ext4    defaults    0       1
UUID=def456-...   /boot/efi        vfat    defaults    0       2
/dev/sdb1         /mnt/backup      ext4    defaults    0       0
//192.168.1.100/share  /mnt/nfs    nfs     defaults    0       0
```

Campos del fstab:
1. **Dispositivo**: puede ser una ruta a `/dev`, un UUID (`UUID=...`) o una etiqueta (`LABEL=...`). Usar UUID es la practica recomendada porque no cambia aunque se renombren los dispositivos.
2. **Punto de montaje**: directorio donde se monta el sistema de archivos. Debe existir previamente.
3. **Tipo**: sistema de archivos (ext4, xfs, btrfs, nfs, swap, etc.).
4. **Opciones**: defaults, noatime, ro, noexec, etc. `defaults` equivale a `rw,suid,dev,exec,auto,nouser,async`.
5. **Dump**: indica si el comando `dump` debe hacer backup del sistema de archivos (0=no, 1=si). Usualmente 0.
6. **Pass**: orden de verificacion en el arranque con `fsck`. 0=no verificar, 1=raiz (primero), 2=resto.

Para montar todos los sistemas de archivos definidos en fstab sin reiniciar:

```bash
mount -a
```

> Una entrada incorrecta en `/etc/fstab` puede impedir que el sistema arranque correctamente. Antes de editar este archivo en un sistema remoto, conviene tener acceso fisico o una consola de recuperacion. Siempre se puede arrancar en modo monousuario para reparar un fstab danado.

## Montaje de dispositivos extraibles

Los sistemas de escritorio modernos montan automaticamente los dispositivos extraibles mediante `udisks` o `systemd-mount`. Sin embargo, en servidores o sistemas sin entorno grafico, el montaje es manual:

```bash
mkdir -p /mnt/usb
mount /dev/sdb1 /mnt/usb
```

Para montar un dispositivo con un sistema de archivos Windows (NTFS) con permisos de lectura/escritura:

```bash
mount -t ntfs-3g -o uid=1000,gid=1000,umask=022 /dev/sdb1 /mnt/usb
```

## Visualizacion del espacio en disco: df

`df` (disk free) muestra el espacio ocupado y disponible en los sistemas de archivos montados. Las opciones mas utiles:

```bash
df -h          # Tamano legible (human-readable: K, M, G)
df -T          # Incluye el tipo de sistema de archivos
df -h /home    # Informacion de una particion especifica
df -i          # Muestra uso de inodos en lugar de bloques
```

Salida tipica:

```bash
df -hT
Filesystem     Type   Size  Used Avail Use% Mounted on
/dev/sda2      ext4   100G   45G   55G  45% /
/dev/sda1      vfat   500M   12M  488M   3% /boot/efi
tmpfs          tmpfs  3.8G  2.1M  3.8G   1% /run/user/1000
```

La columna `Use%` indica el porcentaje de espacio utilizado. Cuando alcanza el 100%, el sistema puede volverse inestable, especialmente si la particion raiz se llena por completo.

## Medicion del tamano de directorios: du

`du` (disk usage) calcula el espacio ocupado por archivos y directorios de forma recursiva.

```bash
du -h /home/usuario     # Tamano legible
du -sh /home/usuario    # Solo el total (-s, summary)
du -h --max-depth=1 /   # Muestra solo un nivel de profundidad
du -h /var/log | sort -h  # Ordenar por tamano
```

`du` es especialmente util para identificar que directorios estan consumiendo mas espacio:

```bash
du -sh /* 2>/dev/null | sort -h
```

> `df` informa del espacio en el sistema de archivos a nivel de bloque; `du` suma los tamanos de los archivos individualmente. La diferencia entre ambos suele deberse a archivos abiertos que han sido eliminados pero cuyo espacio aun no se ha liberado, o a metadatos del propio sistema de archivos. Si `df` muestra menos espacio disponible que la suma de `du`, un reinicio del servicio o del sistema suele liberar ese espacio retenido.

## Montaje de imagenes ISO

Las imagenes ISO se montan como dispositivos de bucle (loopback), sin necesidad de grabarlas fisicamente:

```bash
mount -o loop imagen.iso /mnt/iso
```

El kernel crea un dispositivo `/dev/loop0` y asocia la imagen como si fuera un disco fisico. Esta tecnica es util para inspeccionar contenido de discos de instalacion o distribuciones live sin extraerlas.

## UUID y LABEL

Cada particion formateada recibe un identificador unico universal (UUID) y puede tener una etiqueta (LABEL) definida por el usuario. Estos identificadores son preferibles a las rutas `/dev/sdX` porque permanecen estables incluso cuando cambia el orden de deteccion de discos.

```bash
blkid               # Listar UUID y LABEL de todos los dispositivos
lsblk -f            # Vista arbol con UUID, tipo y punto de montaje
```

Ejemplo de entrada en fstab usando UUID:

```bash
UUID=4f3e2c1a-8b7d-4e6f-9a1c-2d3e4f5a6b7c / ext4 defaults 0 1
```

> Los UUID son especialmente importantes en sistemas con multiples discos, donde la asignacion de nombres `/dev/sda`, `/dev/sdb` puede variar entre reinicios si se conectan o desconectan dispositivos SATA.

