# Configuracion inicial

Despues de completar la instalacion minima, el sistema esta funcional pero no esta preparado para operar de forma fiable en un entorno de servidor. Los valores por defecto del instalador (nombre del equipo, zona horaria, idioma, teclado) estan configurados para el momento de la instalacion, no para la operacion continua. Ademas, el sistema necesita actualizarse para incorporar los parches de seguridad publicados desde la creacion de la imagen de instalacion. Estas configuraciones iniciales parecen triviales, pero tienen implicaciones directas en el correcto funcionamiento de servicios criticos como bases de datos, sistemas de log y aplicaciones que dependen de la marca temporal.

## Nombre del host (hostname)

El hostname identifica al servidor en la red local y en muchos protocolos. Un nombre descriptivo y consistente facilita el monitoreo, los logs y la administracion remota. En lugar de nombres genericos como `localhost` o `server1`, se recomienda un esquema de nombrado que refleje la funcion, el entorno y la ubicacion del servidor: `webprod01`, `dbstaging02`, `monitoring01`.

El hostname se configura en `/etc/hostname` y debe mantenerse sincronizado con `/etc/hosts` para que las resoluciones locales funcionen correctamente. Un hostname mal configurado puede causar problemas con servicios que dependen de la resolucion de nombres, como bases de datos replicadas o sistemas de monitoreo que usan el hostname para identificar nodos.

```bash
# Configurar el hostname permanentemente
sudo hostnamectl set-hostname webprod01

# Verificar
hostnamectl
```

## Zona horaria (timezone)

La zona horaria determina como el sistema interpreta y registra las marcas temporales. Un servidor con la zona horaria incorrecta registra logs con horas distintas a las reales, lo que complica enormemente la correlacion de eventos en incidentes de seguridad o diagnosticos de fallos. Ademas, tareas programadas mediante `cron` o `systemd timers` se ejecutaran en la hora del sistema, lo que podria causar que backups, rotaciones de logs o tareas de mantenimiento se ejecuten en momentos inapropiados.

```bash
# Verificar zona horaria actual
timedatectl

# Configurar zona horaria (ejemplo: Europa/Madrid)
sudo timedatectl set-timezone Europe/Madrid

# Verificar que UTC y la zona local son coherentes
timedatectl
```

> Si el servidor forma parte de un cluster distribuido geograficamente, es recomendable que todos los nodos estén configurados con UTC y que la conversion a hora local se maneje en la capa de aplicacion. Esto evita ambiguedades con horarios de verano/invierno y facilita la comparacion de timestamps entre regiones.

## Locale

El locale determina el idioma, la codificacion de caracteres y el formato de numeros, fechas y monedas. En un servidor, el locale mas comun es `C` o `UTF-8` sin preferencia de idioma, porque los logs y las respuestas de servicios no deben contener caracteres localizados que dificulten el parsing. Un locale mal configurado puede causar problemas con aplicaciones que procesan texto, como servidores web que gestionan caracteres especiales o herramientas de backup que manejan nombres de archivos con tildes.

```bash
# Verificar locales disponibles
locale -a

# Configurar locale por defecto
sudo dpkg-reconfigure locales

# O directamente generar el locale deseado
sudo locale-gen en_US.UTF-8 es_ES.UTF-8
sudo update-locale LANG=en_US.UTF-8
```

## Distribucion de teclado (keyboard layout)

La configuracion del teclado afecta exclusivamente a la sesion local y a la consola de emergencia. Para servidores gestionados remotamente via SSH, la configuracion del teclado del cliente es la que determina como se interpretan las pulsaciones. Sin embargo, si el servidor tiene una consola fisica o una consola de gestion remota (IPMI, iDRAC, iLO), un layout incorrecto puede hacer imposible introducir contraseñas o comandos con caracteres especiales.

```bash
# Configurar teclado (ejemplo: layout español)
sudo dpkg-reconfigure keyboard-configuration

# Aplicar a la consola
sudo setupcon
```

## Actualizaciones del sistema

Una instalacion reciente contiene paquetes con versiones congeladas en el momento de crear la imagen de instalacion. Algunos de esos paquetes pueden tener vulnerabilidades conocidas publicadas despues de la creacion de la imagen. Ejecutar una actualizacion completa antes de poner el servidor en produccion es obligatorio, no opcional.

```bash
# Actualizar lista de paquetes
sudo apt update

# Actualizar todos los paquetes instalados
sudo apt upgrade -y

# En distribuciones RHEL/Rocky
sudo dnf update -y
```

### Actualizaciones automaticas

Para mantener el servidor actualizado a largo plazo, se pueden configurar actualizaciones de seguridad automaticas. En Debian y Ubuntu, `unattended-upgrades` permite descargar e instalar automaticamente los parches de seguridad sin intervencion manual. Esto es especialmente valioso para servidores que no tienen un equipo de administracion dedicado vigilando constantemente los boletines de seguridad.

En entornos de produccion criticos, las actualizaciones automaticas pueden ser demasiado arriesgadas. En esos casos, se sigue un proceso manual con pruebas previas en un entorno de staging, lo que garantiza que una actualizacion no rompa la aplicacion.

> Las actualizaciones de seguridad no son las unicas que importan. Las actualizaciones de funcionalidad (minor updates) a veces corrigen bugs que afectan a la estabilidad. La politica ideal es aplicar actualizaciones de seguridad inmediatamente y programar las demas con regularidad en ventanas de mantenimiento.

## Configuracion del reloj del sistema (NTP)

Un servidor sincroniza su reloj con servidores NTP (Network Time Protocol) para mantener la hora precisa. La precision temporal es critica para la validez de los certificados TLS (que tienen fechas de expiracion), para la correlacion de logs entre multiples servidores y para el correcto funcionamiento de protocolos como Kerberos, que rechazan autenticaciones con desviaciones temporales superiores a cinco minutos.

```bash
# En sistemas con systemd
sudo timedatectl set-ntp true

# Verificar sincronizacion
timedatectl status
```

