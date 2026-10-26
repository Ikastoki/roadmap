# Instalacion minima

Un servidor en produccion no necesita un entorno de escritorio, un navegador web, una suite ofimatica ni ninguno de los componentes que hacen un sistema operativo "amigable" para el uso personal. Cada paquete instalado representa una superficie potencial de ataque: un servicio que podria tener vulnerabilidades, un binario que podria ser explotado, una dependencia que podria introducir conflictos. La instalacion minima es el principio de reducir activamente lo que esta presente en el sistema para que haya menos que defender, menos que mantener y menos que puede fallar.

## Filosofia de la instalacion minima

El concepto no se limita a "no instalar el escritorio grafico". Implica una revision deliberada de cada componente que se incluye en el sistema. Un servidor minimo contiene solo lo necesario para: ejecutar los servicios para los que fue concebido, mantenerse actualizado y seguro, permitir el acceso administrativo remoto. Todo lo demas se omite y se instala posteriormente si se demuestra estrictamente necesario.

Este enfoque tiene consecuencias practicas directas. Un sistema con menos paquetes tiene un tiempo de arranque mas rapido porque hay menos demonios que iniciar. El consumo de memoria RAM en reposo es significativamente menor. El escaneo de vulnerabilidades produce menos resultados falsos positivos. Las actualizaciones de seguridad son mas rapidas porque hay menos paquetes que parchear.

## Seleccion de paquetes durante la instalacion

Los instaladores de las principales distribuciones ofrecen perfiles de instalacion. En Debian, el instalador incluye opciones como "standard system utilities" que instala un conjunto base razonable, pero omite "GNOME", "KDE" y otros metapaquetes de escritorio. En Ubuntu Server, el instalador ya viene optimizado para servidores y no incluye escritorio grafico por defecto, aunque ofrece la posibilidad de instalar snaps de servidor durante la instalacion.

La decision critica es resistir la tentacion de instalar "por si acaso" herramientas que no se van a usar inmediatamente. Si un servidor va a ejecutar un servidor web, no necesita `gcc` ni `make` a menos que se vaya a compilar software en el propio sistema. Si va a ser un nodo de bases de datos, no necesita `apache2`. Cada herramienta se instala bajo demanda, no preventivamente.

> En Debian, durante la instalacion se pueden deseleccionar todos los "software selection" excepto "SSH server" y "standard system utilities" para obtener un sistema verdaderamente minimo. En Ubuntu Server, el instalador ya implementa este enfoque por defecto.

## Particionado seguro

### Por que separar particiones

El particionado de un servidor no sigue la logica de un escritorio donde un unico volumen para todo es suficiente. Un servidor necesita separacion porque diferentes directorios tienen patrones de uso distintos y riesgos diferentes. Si `/var` (donde se almacenan logs, colas de correo y bases de datos) crece indefinidamente, no debe poder llenar la particion donde esta el sistema operativo y dejarlo inoperable. Si `/tmp` recibe archivos de procesos temporales, su crecimiento no debe afectar a otros directorios.

### Esquema de particionado recomendado

Un esquema tipico para un servidor incluye las siguientes particiones:

**`/` (raiz)**: Contiene el sistema operativo, los binarios del sistema y la configuracion base. Debe ser suficientemente grande para el SO y sus actualizaciones (20-40 GB en la mayoria de los casos) pero no necesita ser enorme. Si esta separada de `/home` y `/var`, el agotamiento de espacio en otras particiones no afecta al sistema operativo.

**`/home`**: En un servidor, `/home` es relevante solo si hay usuarios con directorios personales (por ejemplo, usuarios de SSH). En servidores dedicados a un solo servicio, a menudo se omite esta particion o se monta desde un recurso compartido de red.

**`/var`**: Esta particion merece atencion especial. Los logs del sistema (`/var/log`), las bases de datos (`/var/lib/mysql` o `/var/lib/postgresql`), las colas de correo (`/var/mail`) y los archivos temporales de servicios web (`/var/www`) residen aqui. En un servidor de bases de datos, `/var` puede necesitar cientos de gigabytes. En un servidor web, los logs pueden crecer rapidamente si no se rotan correctamente.

**`/tmp`**: Separar `/tmp` permite aplicar opciones de montaje restrictivas como `noexec`, `nosuid` y `nodev` que impiden la ejecucion de binarios desde el directorio temporal. Esto es una medida de seguridad que dificulta la explotacion de vulnerabilidades que dependen de escribir y ejecutar archivos en `/tmp`.

**`swap`**: Aunque modernamente se prefiere el uso de un archivo de swap (`/swapfile`) sobre una particion dedicada, en servidores con mucho consumo de memoria o con requisitos de hibernacion, una particion de swap dedicada ofrece un rendimiento predecible. El tamano depende de la RAM disponible: la regla clasica de "el doble de la RAM" es conservadora; en servidores con 16 GB o mas de RAM, un swap de 4-8 GB suele ser suficiente como buffer de seguridad.

### Sistema de archivos

La eleccion del sistema de archivos afecta tanto la integridad de datos como el rendimiento. `ext4` es el estandar por defecto en la mayoria de distribuciones y ofrece un equilibrio probado entre fiabilidad y rendimiento. `XFS` es preferido para archivos grandes y volumenes de gran tamano (bases de datos, almacenamiento de multimedia). En contextos de contenedores, `Btrfs` o `ZFS` ofrecen snapshotes que permiten rollback rapido de cambios del sistema.

```bash
# Ejemplo simplificado de particionado desde el instalador Debian
# /dev/sda1  -  /boot    -  512MB   - ext4
# /dev/sda2  -  /        -  30GB    - ext4
# /dev/sda3  -  /var     -  50GB    - ext4
# /dev/sda4  -  swap     -  8GB     - swap
```

> No existe un esquema de particionado universalmente correcto. El tamano y la estructura dependen del servicio que ejecutara el servidor. Un servidor de logs necesita una `/var` grande; un servidor de contenedores necesita espacio en `/var/lib/docker`; un servidor de archivos necesita grandes volumenes montados en puntos de montaje dedicados.

## Unidades de sistema (systemd)

Una instalacion minima moderna incluye `systemd` como sistema de init. Esto implica que el gestor de arranque (generalmente GRUB) carga el kernel, que a su vez inicia `systemd`, el cual se encarga de levantar los servicios restantes. En un sistema minimo, el numero de servicios que systemd gestiona es reducido, lo que simplifica el diagnostico de problemas de arranque y reduce los puntos potenciales de fallo.

