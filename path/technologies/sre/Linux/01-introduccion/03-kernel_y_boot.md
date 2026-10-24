# Kernel, shell y proceso de arranque

Entre el hardware y las aplicaciones del usuario existen dos capas fundamentales que determinan como se comporta el sistema: el kernel, que gestiona los recursos fisicos y virtuales, y la shell, que proporciona el interfaz a traves del cual el usuario se comunica con el sistema operativo. Comprender como interactuan ambas, y como se llega desde que se pulsa el boton de encendido hasta que aparece el prompt, es esencial para diagnosticar problemas y entender el funcionamiento interno de Linux.

## El kernel como intermediario

El kernel es el nucleo del sistema operativo. Se ejecuta en un espacio de memoria privilegiado (anillo 0 en la arquitectura x86) y tiene acceso completo al hardware: CPU, memoria RAM, discos, dispositivos de entrada/salida, red. Ningun otro programa puede acceder al hardware directamente; cuando una aplicacion necesita leer un archivo, enviar un paquete de red o asignar memoria, debe solicitar la operacion al kernel a traves de una llamada al sistema (syscall).

Linux es un kernel monolitico: todos los servicios principales (gestion de procesos, memoria, sistema de archivos, drivers de dispositivo) se ejecutan en el mismo espacio de direcciones del kernel. Esto contrasta con los microkernels (como Minix), donde la mayoria de los servicios se ejecutan como procesos de usuario. La ventaja del diseno monolitico es el rendimiento, ya que no hay cambio de contexto entre el kernel y los servicios. La desventaja es que un error en cualquier subsitema puede comprometer todo el sistema.

El kernel Linux soporta mas de 30 arquitecturas hardware, desde mainframes IBM Z hasta relojes inteligentes Android. Su codigo fuente se organiza en arboles de directorios:
- `arch/`: codigo especifico de cada arquitectura
- `drivers/`: controladores de dispositivos
- `fs/`: implementaciones de sistemas de archivos
- `net/`: pila de red
- `kernel/`: gestion de procesos y planificacion
- `mm/`: gestion de memoria

## La shell como interfaz

La shell es un interprete de comandos que actua como capa entre el usuario y el kernel. Su funcion es leer comandos, interpretarlos, ejecutar los programas correspondientes y mostrar los resultados. Aunque existen shells graficas (gestores de ventanas, entornos de escritorio), el termino "shell" en el contexto Unix se refiere casi siempre a la interfaz de linea de comandos.

Bash (Bourne Again SHell) es la shell predeterminada en la mayoria de las distribuciones. Es compatible con la sintaxis de la shell Bourne original (sh) e incorpora mejoras como historial de comandos, autocompletado, correccion de nombres de archivos y programacion estructurada. Bash es un proyecto GNU y se distribuye bajo la GPL.

Zsh (Z shell) es una shell moderna que extiende Bash con caracteristicas adicionales: temas personalizables, resaltado sintactico, correccion ortografica automatica, busqueda en el historial mediante patrones y un potente sistema de plugins. Oh My Zsh es un framework que simplifica la configuracion de Zsh y proporciona cientos de temas y plugins.

Fish (Friendly Interactive SHell) se disena desde cero con el enfoque de ser usable sin configuracion. Ofrece resaltado sintactico inmediato, sugerencias de comandos basadas en el historial y autocompletado predictivo. Su sintaxis de scripting difiere de POSIX, lo que la hace menos portable pero mas legible.

La relacion entre la shell y la terminal es sutil pero importante. La terminal (o emulador de terminal) es el programa que dibuja la ventana y maneja la entrada de teclado y la salida de texto. La shell es el interprete que se ejecuta dentro de esa terminal. GNOME Terminal, Konsole, iTerm2 y Alacritty son emuladores de terminal; Bash, Zsh y Fish son shells.

## Proceso de arranque

Cuando se enciende un ordenador, el procesador ejecuta el firmware almacenado en la placa base. Este firmware sigue el estandar UEFI (Unified Extensible Firmware Interface), que ha reemplazado al antiguo BIOS en la mayoria de los sistemas modernos. UEFI inicializa el hardware, realiza el POST (Power-On Self Test) y busca un gestor de arranque en la particion ESP (EFI System Partition).

El gestor de arranque mas comun en Linux es GRUB (Grand Unified Bootloader). GRUB lee su configuracion desde `/boot/grub/grub.cfg`, presenta un menu con los kernels disponibles (y opcionalmente otros sistemas operativos), y carga el kernel y el initramfs en memoria. GRUB puede acceder a sistemas de archivos (ext4, XFS, Btrfs) directamente, lo que le permite leer el kernel desde cualquier particion soportada.

El initramfs (initial RAM filesystem) es una imagen que contiene los modulos del kernel necesarios para montar el sistema de archivos raiz. Como el kernel no puede leer el sistema de archivos real si necesita un controlador que aun no esta cargado (por ejemplo, un modulo para el controlador NVMe), el initramfs proporciona un sistema de archivos temporal en RAM con los modulos esenciales. El kernel ejecuta `/init` dentro del initramfs, que carga los modulos, descubre los dispositivos de bloque y monta la raiz real.

Una vez montada la raiz, el kernel ejecuta el primer proceso de espacio de usuario: el init system. En las distribuciones modernas, este proceso es systemd, aunque todavia se utiliza SysV init en algunos sistemas embebidos o minimalistas. systemd es responsable de montar los sistemas de archivos restantes (segun `/etc/fstab`), iniciar los servicios configurados, gestionar los targets de arranque (multi-user.target, graphical.target) y lanzar la pantalla de inicio de sesion.

```bash
# Ver el gestor de arranque instalado
grub-install --version
efibootmgr -v  # (en sistemas UEFI)

# Configurar GRUB tras cambios en el kernel
update-grub   # (Debian/Ubuntu)
grub2-mkconfig -o /boot/grub2/grub.cfg  # (Fedora/RHEL)
```

## Parametros del kernel

Durante el arranque, GRUB puede pasar parametros al kernel para modificar su comportamiento. Estos parametros se anaden a la linea de arranque en el menu de GRUB o se configuran permanentemente en `/etc/default/grub` mediante la variable `GRUB_CMDLINE_LINUX`.

Parametros comunes:
- `quiet`: suprime la mayoria de los mensajes del kernel durante el arranque.
- `splash`: muestra una pantalla de bienvenida grafica.
- `nomodeset`: desactiva la conmutacion de modos KMS, util para problemas con drivers graficos.
- `acpi=off`: desactiva ACPI, necesario en hardware con controladores ACPI problematicos.
- `mem=4G`: limita la memoria visible a 4 GB, util para pruebas.
- `single` o `1`: arranca en modo monousuario (mantenimiento).
- `root=/dev/sda2`: especifica manualmente la particion raiz.

```bash
# Ver los parametros con los que arranco el kernel actual
cat /proc/cmdline

# Configurar parametros permanentes (Debian/Ubuntu)
# Editar /etc/default/grub y luego ejecutar:
update-grub
```

## Modulos del kernel

Linux es modular: los controladores y las funcionalidades se compilan como modulos que pueden cargarse y descargarse en caliente sin reiniciar el sistema. Esto permite mantener un kernel pequeno en memoria y solo cargar los controladores necesarios para el hardware detectado.

```bash
# Listar modulos cargados actualmente
lsmod

# Cargar un modulo
modprobe <nombre_del_modulo>

# Descargar un modulo
modprobe -r <nombre_del_modulo>

# Informacion de un modulo
modinfo <nombre_del_modulo>

# Listar modulos disponibles en el sistema
ls /lib/modules/$(uname -r)/
```

El archivo `/etc/modprobe.d/` contiene configuraciones que modifican el comportamiento de modprobe. Los blacklists son especialmente utiles para evitar que un modulo se cargue automaticamente:

```bash
# /etc/modprobe.d/blacklist.conf
blacklist nouveau       # Desactiva el driver Nouveau
blacklist pcspkr        # Silencia el altavoz interno
```

> Los modulos residen en `/lib/modules/$(uname -r)/`. Cada version del kernel tiene su propio directorio de modulos. Si se compila un kernel personalizado, debe ejecutarse `make modules_install` para poblarlo.

## Actualizacion y compilacion del kernel

En las distribuciones modernas, el kernel se actualiza como cualquier otro paquete. El gestor de paquetes descarga la nueva imagen, instala los modulos correspondientes y ejecuta el script de postinstalacion que regenera la configuracion de GRUB. Tras un reinicio, el nuevo kernel esta activo.

```bash
# Versiones del kernel instaladas
ls /boot/vmlinuz-*

# Kernel actual en ejecucion
uname -r

# En Debian/Ubuntu, instalar un kernel
apt install linux-image-generic linux-headers-generic
```

Compilar el kernel manualmente es un proceso que solo se justifica en escenarios especificos: kernels minimizados para sistemas empotrados, parches de seguridad que no han llegado a la distribucion, o experimentacion academica.

```bash
# Compilacion basica del kernel
tar xf linux-6.x.tar.xz
cd linux-6.x
make menuconfig     # Configurar opciones
make -j$(nproc)     # Compilar
make modules_install
make install
```

La mayoria de los usuarios jamas necesitara compilar un kernel. Las distribuciones proporcionan kernels genericos que funcionan en la practica totalidad del hardware. La compilacion manual sigue siendo valiosa como ejercicio de aprendizaje.

## Diagnostico de problemas de arranque

Cuando el sistema no arranca, la informacion esta disponible aunque la pantalla se quede en negro. El primer paso es examinar los mensajes del kernel:

```bash
# Mensajes del kernel del arranque actual
dmesg | less

# Mensajes del arranque anterior (si systemd-journald)
journalctl -k -b -1

# Ver el log del arranque completo
journalctl -b
```

Problemas frecuentes y su diagnostico:
- **Kernel panic**: el kernel encuentra una condicion irrecuperable. El mensaje incluye una pila de llamadas (call trace) que senala el modulo o la funcion que fallo.
- **Initramfs corrupto**: el sistema se queda en "Gave up waiting for root device". Suele resolverse regenerando el initramfs con `update-initramfs -u` o `dracut --force`.
- **Error de modulo grafico**: pantalla negra tras cargar el kernel. Arrancar con `nomodeset` permite acceder al sistema y desinstalar o reconfigurar el driver.
- **Root filesystem no montado**: aparece un shell de emergencia. Verificar `cat /proc/partitions` y montar manualmente la particion raiz.

> systemd-bootchart genera un grafico SVG del tiempo de inicio de cada servicio, util para identificar cuellos de botella en el arranque. Tambien puede ejecutarse `systemd-analyze blame` para listar servicios ordenados por tiempo de inicio.

