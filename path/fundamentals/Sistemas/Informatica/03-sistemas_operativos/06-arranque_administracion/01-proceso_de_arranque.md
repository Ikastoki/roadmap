# Proceso de arranque

El proceso de arranque (*boot process*) es la secuencia de eventos que transforma un equipo apagado en un sistema operativo funcional y listo para recibir peticiones del usuario o de la red. Comprender el arranque es esencial para diagnosticar problemas de inicio, configurar dual boot o personalizar el comportamiento del sistema.

El arranque de Linux se divide en varias fases claramente diferenciadas: desde la ejecucion del firmware (BIOS/UEFI) hasta la ejecucion del primer proceso del usuario (`init` o `systemd`). Cada fase tiene un rol especifico y puntos de fallo identificables.

## Fase 1: firmware (BIOS/UEFI)

Cuando se enciende el equipo, el procesador ejecuta el codigo del firmware almacenado en un chip ROM/flash de la placa base. Este firmware realiza la verificacion de hardware (*POST - Power-On Self-Test*) y busca un dispositivo de arranque valido.

**BIOS (Basic Input/Output System):** firmware tradicional que busca dispositivos de arranque en un orden predefinido (HDD, SSD, USB, red). Busca una tabla de particiones MBR en el primer sector del disco y ejecuta el codigo de arranque contenido ahi.

**UEFI (Unified Extensible Firmware Interface):** firmware moderno que busca una particion EFI System Partition (ESP) formateada en FAT32 y contiene un archivo ejecutable (`.efi`) que corresponde al gestor de arranque. UEFI soporta GPT (GUID Partition Table) que permite discos mayores de 2 TB y mas de 4 particiones primarias.

```bash
# Verificar modo de arranque
ls /sys/firmware/efi  # Si existe, es UEFI; si no, es BIOS/CSM

# Ver la tabla de particiones GPT
sudo gdisk -l /dev/sda

# Ver la particion ESP
sudo fdisk -l /dev/sda | grep "EFI System"
```

## Fase 2: bootloader (GRUB 2)

El *bootloader* es el programa que carga el kernel de Linux en memoria y le pasa los parametros de arranque. En la mayoria de las distribuciones modernas, GRUB 2 es el gestor de arranque predeterminado.

GRUB 2 busca su configuracion en `/boot/grub/grub.cfg` (generado automaticamente) y muestra un menu de opciones de arranque. El usuario puede seleccionar el kernel, modificar los parametros de arranque en tiempo real o elegir un kernel anterior si el mas reciente falla.

```bash
# Editar configuracion de GRUB (archivos fuente)
sudo vim /etc/default/grub
sudo vim /etc/grub.d/40_custom

# Regenerar grub.cfg despues de cambios
sudo update-grub    # Debian/Ubuntu
sudo grub2-mkconfig -o /boot/grub2/grub.cfg    # RHEL/CentOS

# Reinstalar GRUB en el MBR o EFI
sudo grub-install /dev/sda
```

Los parametros de arranque criticos incluyen:

- `root=/dev/sdaX`: particion raiz del sistema de archivos.
- `ro`: montar la raiz como solo lectura inicialmente.
- `quiet splash`: ocultar mensajes de arranque (modo grafico).
- `single` o `init=/bin/bash`: arrancar en modo monousuario para recuperacion.

## Fase 3: kernel y initramfs

El kernel (`/boot/vmlinuz-*`) se carga en memoria y ejecuta su codigo de inicializacion. Primero necesita montar el sistema de archivos raiz, pero si los drivers necesarios (controlador SATA, NVMe, etc.) estan como modulos, el kernel no puede montarlos directamente.

La solucion es el **initramfs** (*initial RAM filesystem*), una imagen comprimida que contiene los modulos del kernel y un script de inicializacion. El initramfs se carga en memoria junto con el kernel y ejecuta `/init` (o `/sbin/init` del initramfs) que:

1. Carga los modulos necesarios para el hardware detectado.
2. Monta el sistema de archivos raiz real.
3. Ejecuta `switch_root` para cambiar la raiz al sistema de archivos real.
4. Ejecuta `/sbin/init` del sistema real.

```bash
# Reconstruir initramfs
sudo update-initramfs -u    # Debian/Ubuntu
sudo dracut -f              # RHEL/CentOS

# Ver contenido del initramfs
lsinitramfs /boot/initrd.img-* | head -30
```

## Fase 4: systemd / init

Una vez que el kernel ha montado el sistema de archivos raiz y ha ejecutado `init` (PID 1), comienza la inicializacion de los servicios del sistema. En los sistemas modernos, `systemd` es el gestor de init predominante.

systemd arranca paralelamente los servicios que no tienen dependencias entre si, reduciendo significativamente el tiempo de arranque frente al init tradicional (SysVinit), que ejecutaba los scripts de inicio secuencialmente.

```bash
# Ver el grafico de arranque
systemd-analyze

# Ver los servicios mas lentos
systemd-analyze blame | head -10

# Ver dependencias de un servicio
systemd-analyze critical-chain nginx.service

# Ver tiempo de arranque del kernel vs userspace
systemd-analyze time
```

## Modo de recuperacion

Cuando el sistema no puede arranque correctamente, el modo de recuperacion permite solucionar problemas:

1. Reiniciar y seleccionar "Advanced options" en el menu de GRUB.
2. Elegir el kernel con "(recovery mode)".
3. El sistema arranca en modo monousuario con la raiz montada como solo lectura.
4. Opciones disponibles: reparar el sistema de archivos (`fsck`), revisar logs, cambiar contrasena de root, reinstalar GRUB.

```bash
# En modo recuperacion
fsck -y /dev/sda1        # Reparar sistema de archivos
mount -o remount,rw /    # Remontar raiz como lectura/escritura
dpkg --configure -a      # Reparar paquetes rotos (Debian/Ubuntu)
```

> El arranque es la fase mas vulnerable del sistema: un error en GRUB, un kernel corrupto o un initramfs inadecuado pueden dejar el sistema inutilizable. Mantener varios kernels instalados y conocer el proceso de recuperacion es esencial para cualquier administrador de sistemas.

