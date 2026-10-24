# Estructura del sistema de archivos FHS

El sistema de archivos en Linux no es un simple arbol de carpetas sin criterio. Sigue el estandar FHS (Filesystem Hierarchy Standard), una especificacion que define la disposicion de los directorios en sistemas Unix-like. Este estandar garantiza que tanto el sistema operativo como las aplicaciones de terceros puedan localizar archivos de configuracion, binarios, bibliotecas y datos variables sin ambiguedad. La existencia del FHS es lo que permite que un script escrito en Debian funcione igual en Arch o Fedora en lo que respecta a las rutas de recursos del sistema.

## Directorios raiz y su proposito

El directorio raiz `/` es la cima de la jerarquia. Todos los demas directorios cuelgan de el, directa o indirectamente. A diferencia de Windows, donde existen varias raices (C:, D:), Linux presenta una unica raiz desde la que se montan todos los dispositivos de almacenamiento.

- `/` (raiz): la base del sistema de archivos. Contiene los directorios esenciales para el arranque y el funcionamiento basico. Solo el superusuario tiene permiso de escritura en este nivel.
- `/home`: contiene los directorios personales de los usuarios. Cada usuario tiene una carpeta (`/home/usuario`) donde almacena sus documentos, configuraciones de usuario y archivos personales. En sistemas multiusuario, esta separacion es fundamental para el aislamiento de datos.
- `/etc`: alberga los archivos de configuracion del sistema. Archivos como `passwd`, `shadow`, `fstab`, `hosts` y `ssh/sshd_config` residen aqui. Es el directorio que un administrador modifica con mas frecuencia.
- `/var`: contiene datos variables: logs del sistema (`/var/log`), bases de datos, colas de impresion, cache de paquetes (`/var/cache/apt`) y archivos temporales persistentes entre reinicios.
- `/usr`: el directorio de recursos compartidos de usuario. Alberga binarios, bibliotecas, documentacion y codigo fuente. Tradicionalmente era el directorio donde se instalaban los programas del usuario, pero en sistemas modernos aloja la mayoria del software del sistema.
- `/bin`: binarios esenciales del sistema necesarios durante el arranque y en modo monousuario. Comandos como `ls`, `cp`, `mv`, `bash` residen aqui. Tradicionalmente era un directorio independiente.
- `/sbin`: binarios esenciales de administracion del sistema (`fdisk`, `mkfs`, `mount`).
- `/opt`: software opcional de terceros. Aplicaciones que no forman parte de la distribucion se instalan aqui, cada una en su propio subdirectorio (ej. `/opt/google/chrome`). Esta convencion evita la contaminacion del arbol del sistema.
- `/tmp`: archivos temporales. Cualquier usuario puede escribir aqui, pero los archivos se eliminan al reiniciar el sistema (o periodicamente mediante mecanismos como `systemd-tmpfiles`).
- `/dev`: archivos de dispositivos. Linux trata los dispositivos hardware como archivos especiales. Discos (`sda`, `nvme0n1`), terminales (`tty`), dispositivos de audio (`audio`) y otros aparecen aqui como nodos.
- `/proc`: sistema de archivos virtual que expone informacion del kernel y de los procesos en tiempo real. No ocupa espacio en disco; es una interfaz con el kernel.
- `/sys`: similar a `/proc`, pero enfocado en la informacion de dispositivos y controladores. Es la representacion del modelo de dispositivos del kernel.
- `/run`: archivos de estado de procesos en ejecucion. Sustituye a `/var/run` en sistemas modernos con systemd.
- `/lib`: bibliotecas compartidas esenciales necesarias por los binarios de `/bin` y `/sbin`.
- `/mnt` y `/media`: puntos de montaje temporales. `/media` se usa tipicamente para dispositivos extraibles montados automaticamente; `/mnt` para montajes manuales temporales.

## Diferencia entre /bin y /usr/bin

La distincion entre `/bin` y `/usr/bin` tiene origenes historicos. En los sistemas Unix primitivos, `/bin` contenia los binarios minimos para arrancar el sistema y `/usr/bin` albergaba el resto de programas, montado desde una particion separada. Con el tiempo, esta separacion perdio sentido, pero la inercia y el estandar la mantuvieron.

En las distribuciones modernas, especialmente aquellas que adoptan `systemd` y `usrmerge`, `/bin` es un enlace simbolico a `/usr/bin`:

```bash
ls -l /bin
lrwxrwxrwx 1 root root 7 jul  1  2024 /bin -> usr/bin
```

La migracion conocida como *usrmerge* unifica los directorios: `/bin` -> `/usr/bin`, `/sbin` -> `/usr/sbin`, `/lib` -> `/usr/lib`. Esto simplifica la estructura y elimina duplicidades. A efectos practicos, el administrador puede considerar que todos los binarios residen en `/usr/bin`, y las referencias a `/bin` funcionan gracias al enlace simbolico.

## Tipos de archivos en Linux

En Linux, todo es un archivo. Esta maxima significa que los dispositivos, tuberias, sockets y directorios se representan mediante descriptores de archivo. El comando `ls -l` muestra el tipo de cada entrada mediante el primer caracter de la salida:

| Caracter | Tipo de archivo | Descripcion |
|----------|----------------|-------------|
| `-` | Regular | Archivo comun de datos, texto, binario, etc. |
| `d` | Directorio | Contenedor de entradas del sistema de archivos |
| `l` | Enlace simbolico | Referencia a otro archivo o directorio |
| `c` | Dispositivo de caracter | Dispositivos que transmiten datos byte a byte (terminales, puertos serie) |
| `b` | Dispositivo de bloque | Dispositivos que transmiten datos en bloques (discos, particiones) |
| `p` | Tuberia con nombre (pipe) | Comunicacion entre procesos |
| `s` | Socket | Comunicacion entre procesos en red o local |

### Archivos regulares

Son el tipo mas comun. Contienen datos en cualquier formato: texto plano, binarios ejecutables, imagenes, comprimidos. No existe distincion intrinseca entre un archivo de texto y uno binario; la diferencia la determina el contenido y la aplicacion que lo interpreta.

### Directorios

Un directorio es una tabla que asocia nombres de archivo con numeros de inodo (el identificador unico del archivo en el sistema de archivos). Cuando se crea un archivo, se asigna un inodo y se anade una entrada en el directorio que lo contiene.

```bash
ls -la
drwxr-xr-x  5 usuario usuario  4096 jul  1 12:00 .
drwxr-xr-x  3 usuario usuario  4096 jul  1 11:00 ..
-rw-r--r--  1 usuario usuario  1024 jul  1 12:00 documento.txt
```

Las entradas `.` (directorio actual) y `..` (directorio padre) son enlaces duros especiales presentes en todo directorio.

### Enlaces

Linux soporta dos tipos de enlaces:

- **Enlace duro (hard link)**: multiple entrada de directorio que apunta al mismo inodo. No se puede crear enlaces duros a directorios ni entre sistemas de archivos diferentes. Al eliminar un enlace duro, los datos persisten mientras exista al menos otro enlace.
- **Enlace simbolico (symlink)**: archivo especial que contiene la ruta a otro archivo o directorio. Si se elimina el destino, el enlace se rompe (dangling symlink). Se crean con `ln -s`.

```bash
ln archivo.txt enlace_duro.txt   # Enlace duro
ln -s archivo.txt enlace_simbolico.txt  # Enlace simbolico
```

> Los enlaces duros son indistinguibles del archivo original: ambos son entradas equivalentes que referencian el mismo inodo. Los enlaces simbolicos, en cambio, son entradas independientes que almacenan una ruta.

### Dispositivos

Los dispositivos se representan como archivos especiales en `/dev`. Se dividen en dos categorias:

- **Dispositivos de caracter (`c`)**: transmiten datos secuencialmente, un byte a la vez. Ejemplos: `/dev/tty` (terminal), `/dev/random` (generador aleatorio).
- **Dispositivos de bloque (`b`)**: transfieren datos en bloques de tamano fijo, permitiendo acceso aleatorio. Ejemplos: `/dev/sda` (disco), `/dev/nvme0n1` (SSD NVMe).

Los dispositivos se identifican por un numero *major* (tipo de dispositivo) y *minor* (instancia concreta):

```bash
ls -l /dev/sda
brw-rw---- 1 root disk 8, 0 jul  1 12:00 /dev/sda
```

`8, 0` indica major=8 (dispositivos SCSI/SATA) y minor=0 (primer disco).

