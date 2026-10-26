# Herramientas esenciales

Un servidor minimo instalado desde cero carece de muchas de las herramientas que un administrador necesita para diagnostica, mantenimiento y operacion diaria. La instalacion minima solo incluye lo basico del sistema, pero la gestion real del servidor requiere un conjunto de utilidades que permiten transferir archivos, monitorear el rendimiento, gestionar el codigo fuente, configurar el cortafuegos y editar configuraciones de forma segura. Estas herramientas no son opcionales: son el EQUIPO BASICO sin el cual administrar un servidor es ineficiente y propenso a errores.

## transferencia de archivos: curl y wget

`curl` y `wget` son herramientas de transferencia de datos a traves de protocolos de red, pero sus enfoques son distintos y complementarios.

**curl** es una libreria y herramienta de transferencia de datos que soporta una enormidad de protocolos: HTTP, HTTPS, FTP, SFTP, SCP, LDAP, POP3, IMAP, SMTP, entre otros. Es la herramienta preferida para probar endpoints REST, descargar archivos con autenticacion compleja o interactuar con APIs de servicios en la nube. Su flexibilidad lo hace indispensable en scripts de automatizacion que necesitan interactuar con servicios web.

```bash
# Descargar un archivo
curl -O https://ejemplo.com/archivo.tar.gz

# Probar un endpoint API
curl -X POST -H "Content-Type: application/json" -d '{"key":"value"}' https://api.ejemplo.com/v1/recurso

# Seguir redirecciones y mostrar headers
curl -L -I https://ejemplo.com
```

**wget** es una herramienta mas simple orientada exclusivamente a la descarga de archivos via HTTP, HTTPS y FTP. Su ventaja sobre curl es la capacidad de hacer descargas recursivas (descargar un directorio completo manteniendo la estructura) y la reanudacion automatica de descargas interrumpidas. Para descargas directas de paquetes, imagenes ISO o backups, wget es mas practico.

```bash
# Descarga con reanudacion
wget -c https://ejemplo.com/archivo-grande.tar.gz

# Descarga recursiva de un sitio web para offline
wget -r -np -nH --cut-dirs=3 https://docs.ejemplo.com/
```

## Editores de texto: vim y nano

Un servidor no tiene interfaz grafica, por lo que la edicion de archivos de configuracion se realiza desde la linea de comandos. Los dos editores principales son `vim` y `nano`, y la eleccion entre ellos depende mas de la familiaridad que de capacidades tecnicas.

**vim** es un editor modal con una curva de aprendizaje pronunciada pero una eficiencia extraordinaria una vez dominado. Sus modos de insercion, comando y visual permiten manipular texto con rapidez. En un servidor, donde la edicion de archivos de configuracion largos es frecuente, la eficiencia de vim se traduce en tiempo ahorrado y menos errores de formato.

**nano** es un editor no modal con una interfaz mas intuitiva que muestra los atajos de teclado en la parte inferior de la pantalla. Para ediciones rapidas o para administradores que no usan vim habitualmente, nano elimina la barrera de entrada. Su limitacion principal es la falta de funcionalidades avanzadas como macros, busqueda con expresiones regulares complejas o edicion multipantalla.

> Lo mas importante es que el editor este instalado y funcione. Un servidor sin editor de texto instalado es inutil para la administracion remota. En algunos sistemas minimos, ni `vim` ni `nano` estan presentes por defecto y deben instalarse explicitamente con `apt install vim` o `apt install nano`.

## Monitorizacion en tiempo real: htop

`htop` es un monitor de procesos interactivo que reemplaza al clasico `htop` (y al menos intuitivo `top`). Muestra en tiempo real el uso de CPU, memoria, swap y procesos activos, pero con una interfaz mas clara y funcionalidades como la capacidad de buscar procesos por nombre, ordenar por multiples criterios y enviar señales a procesos directamente desde la interfaz.

En un servidor, `htop` es la primera herramienta que se ejecuta cuando algo va mal: un servicio no responde, el sistema se ralentiza o la memoria se agota. Verificar rapidamente que procesos estan consumiendo recursos permite diagnosticar si un servicio se ha colgado, si hay un bucle infinito o si el sistema necesita mas recursos.

```bash
# Instalar htop
sudo apt install htop

# Ejecutar
htop
```

Las columnas clave que htop muestra son: `PID` (identificador del proceso), `USER` (usuario que lo ejecuta), `CPU%` (consumo de CPU), `MEM%` (consumo de memoria) y `COMMAND` (comando que inicio el proceso). Un proceso de PostgreSQL consumiendo 100% de CPU puede indicar una consulta mal optimizada; un proceso de Java consumiendo toda la memoria puede indicar un memory leak.

## Control de versiones: git

`git` es el sistema de control de versiones distribuido mas utilizado y su presencia en un servidor va mas alla del desarrollo de software. Los archivos de configuracion de servicios criticos (nginx, apache, docker-compose, kubernetes manifests) se benefician enormemente de ser gestionados con git: cada cambio queda registrado, se puede revertir a versiones anteriores y se puede sincronizar entre multiples servidores.

En un servidor de produccion, git permite mantener un historial de cambios en la configuracion. Si una modificacion en `nginx.conf` causa un fallo, el administrador puede comparar la version actual con la anterior, identificar exactamente que cambio y revertirlo en segundos. Sin git, esta operacion depende de backups manuales o de la memoria del administrador.

```bash
# Instalar git
sudo apt install git

# Configurar identidad (necesaria para commits)
git config --global user.name "Administrador"
git config --global user.email "admin@empresa.local"
```

## Cortafuegos: ufw y firewalld

El cortafuegos es la barrera entre el servidor y el exterior. Sin el, todos los puertos del servidor estan abiertos a cualquier conexion entrante, lo que significa que servicios que solo deberian ser accesibles internamente quedan expuestos. Un cortafuegos bien configurado implementa el principio de "denegar por defecto, permitir explicitamente": todas las conexiones entrantes se bloquean a menos que se permitan explicitamente.

### ufw (Uncomplicated Firewall)

`ufw` es la herramienta de cortafuegos recomendada en Ubuntu y Debian. Su nombre refleja su filosofia: simplificar la gestion de `iptables` o `nftables` con comandos legibles y directos.

```bash
# Instalar ufw
sudo apt install ufw

# Politica por defecto: denegar entradas, permitir salidas
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir SSH (critico para no bloquear el acceso)
sudo ufw allow 22/tcp

# Permitir HTTP y HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Activar el cortafuegos
sudo ufw enable

# Ver reglas activas
sudo ufw status verbose
```

> El error mas comun con cortafuegos es activar las reglas de denegacion sin haber permitido SSH primero, bloqueando el acceso al servidor. Siempre permitir SSH (puerto 22) antes de activar ufw. Si el servidor es accesible via consola fisica o IPMI, se puede recuperar el acceso; si solo es accesible via SSH, un bloqueo de SSH significa reconexion manual.

### firewalld

`firewalld` es el cortafuegos por defecto en RHEL, CentOS y Fedora. A diferencia de ufw, firewalld opera con zonas (public, internal, drop, trusted) que definen el nivel de confianza de cada interfaz de red. Las reglas se asocian a zonas, lo que permite una gestion mas granular en servidores con multiples interfaces de red.

```bash
# Ver zonas activas
sudo firewall-cmd --get-active-zones

# Permitir HTTP en la zona public
sudo firewall-cmd --zone=public --add-service=http --permanent

# Recargar para aplicar cambios
sudo firewall-cmd --reload
```

## Herramientas complementarias

Ademas de las herramientas principales, un servidor necesita un conjunto complementario que completa el equipamiento basico:

**htop** se complementa con `iostat` y `vmstat` para diagnosticar problemas de disco y memoria virtual respectivamente. `netstat` o su sucesor `ss` muestran las conexiones de red activas y los puertos en escucha, lo que es fundamental para verificar que un servicio esta escuchando en el puerto correcto.

**rsync** es la herramienta estandar para sincronizar archivos entre sistemas de forma eficiente, transfiriendo solo las diferencias. Para backups locales o sincronizacion entre servidores, rsync es insustituible por su fiabilidad y eficiencia en el uso de ancho de banda.

**tmux** o `screen` permiten mantener sesiones de terminal activas incluso cuando la conexion SSH se interrumpe. En un servidor donde las tareas de mantenimiento pueden durar horas, perder la sesion por un corte de red no es aceptable.

```bash
# Ver puertos en escucha
sudo ss -tlnp

# Sincronizar directorios con rsync
rsync -avz /var/www/ backup@192.168.1.200:/backup/www/

# Iniciar sesion tmux
tmux new -s mantenimiento
```

> Las herramientas esenciales no se instalan una vez y se olvidan. Deben mantenerse actualizadas junto con el resto del sistema, ya que tambien pueden contener vulnerabilidades. La actualizacion regular de herramientas de administracion es parte del ciclo de mantenimiento del servidor.

