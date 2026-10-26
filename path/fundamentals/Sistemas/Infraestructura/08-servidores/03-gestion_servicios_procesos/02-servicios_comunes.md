# Servicios comunes en servidores Linux

Un servidor Linux tipico ejecuta varios servicios especializados que cubren necesidades de acceso remoto, servimiento de contenido, almacenamiento de datos y automatizacion. Conocer el proposito y comportamiento de cada uno es esencial para mantener la disponibilidad del sistema y diagnosticar problemas cuando algo falla.

## sshd: acceso remoto seguro

El daemon SSH (Secure Shell) es el servicio de acceso remoto por defecto en la mayoria de distribuciones Linux. Permite conectarse a traves de un canal cifrado al servidor, tanto para gestion interactiva como para transferencia de archivos.

Funciona escuchando en el puerto TCP 22 por defecto. La autenticacion puede realizarse mediante contrasenas (desaconsejado en produccion) o claves publicas/privadas, que proporcionan una seguridad significativamente superior. El servidor valida la identidad del cliente verificando que la clave privada coincide con la publica almacenada en `~/.ssh/authorized_keys`.

Para su configuracion, se edita el archivo `/etc/ssh/sshd_config`, donde se definen parametros como el puerto de escucha, los metodos de autenticacion permitidos, los usuarios que pueden conectarse y las opciones de cifrado. Cualquier cambio requiere reiniciar o recargar el servicio (`systemctl reload sshd`).

> sshd es el servicio mas critico en un servidor remoto. Antes de cualquier cambio, es recomendable tener una sesion alternativa abierta (consola fisica o IPMI) para evitar quedarse fuera del sistema.

## nginx: servidor web de alto rendimiento

nginx funciona como servidor web inverso, proxy reverso y balanceador de carga. Su arquitectura basada en eventos le permite manejar miles de conexiones simultaneas con un consumo de memoria muy bajo en comparacion con servidores tradicionales como Apache con su modelo de procesos.

En un escenario tipico, nginx recibe peticiones HTTP/HTTPS en los puertos 80 y 443, sirve archivos estaticos directamente desde el disco y redirige las peticiones dinamicas a un backend como PHP-FPM, Node.js o una aplicacion Python. Esta separacion permite escalar cada componente de forma independiente.

La configuracion se organiza en bloques: `http`, `server` y `location`. Un bloque `server` define un virtual host (que dominio atiende), y cada bloque `location` define que hacer con rutas especificas dentro de ese dominio. Esto permite alojar multiples sitios web en un mismo servidor.

```bash
systemctl status nginx
```

## postgresql: base de datos relacional

PostgreSQL es un sistema de gestion de bases de datos relacional conocido por su cumplimiento estricto del estandar SQL, su soporte para tipos de datos avanzados y su extensibilidad. A diferencia de MySQL, PostgreSQL implementa transacciones ACID completas y tiene un motor de replicacion robusto para configuraciones de alta disponibilidad.

El servicio principal se ejecuta como un conjunto de procesos: un proceso supervisor (`postmaster`) y multiples procesos de trabajo que atienden las conexiones de los clientes. La memoria compartida y los buffers de cache son componentes criticos de su rendimiento, y su configuracion en `postgresql.conf` debe ajustarse segun los recursos disponibles del servidor.

La autenticacion de clientes se controla en `pg_hba_file.conf`, que define que usuarios pueden conectarse desde que direcciones IP y con que metodo de autenticacion. Esto es el primer nivel de seguridad antes de llegar a los permisos de la base de datos.

## docker: contenedorizacion de aplicaciones

Docker transforma la forma en que se despliegan aplicaciones al empaquetar cada servicio junto con todas sus dependencias en un contenedor aislado. Esto elimina el problema clasico de "funciona en mi maquina" al garantizar que el entorno de ejecucion sea identico en desarrollo, pruebas y produccion.

Docker utiliza la arquitectura cliente-servidor: el daemon `dockerd` gestiona imagenes, contenedores y redes, mientras que el cliente CLI envia comandos a traves de una API REST. Las imagenes se construyen a partir de un `Dockerfile` que define las capas del sistema de archivos, y se almacenan en un registro (Docker Hub, registro privado).

En servidores, Docker se usa frecuentemente para ejecutar bases de datos, colas de mensajes, herramientas de monitoreo y aplicaciones web, cada una aislada en su propio contenedor con red y almacenamiento virtual dedicado. Docker Compose permite definir y orquestar multiples contenedores que forman una aplicacion completa.

## cron: tareas periodicas

cron es el demonio encargado de ejecutar tareas Automaticamente en horarios programados. Es el mecanismo clasico para realizar respaldos, limpieza de archivos temporales, rotacion de logs y cualquier operacion que deba ejecutarse periodicamente sin intervencion manual.

Funciona mediante un archivo de planificacion (`crontab`) que define las tareas en formato de texto. Cada linea especifica una fecha/hora de ejecucion y el comando a lanzar. El demonio cron se despierta periodicamente, compara la hora actual con las entradas del crontab y ejecuta las que coincidan.

```bash
crontab -e    # Editar el crontab del usuario actual
crontab -l    # Listar las tareas programadas
```

La sintaxis de cron sigue el formato: `minuto hora dia mes dia_semana comando`, donde se pueden usar valores numericos, rangos, listas y asteriscos para indicar "cada vez".

> Aunque cron sigue siendo universal, systemd timers ofrecen ventajas en entornos modernos como mejor integracion con los logs del sistema y gestion de dependencias.

## Interaccion entre servicios

Estos servicios no operan de forma aislada. Un servidor web tipico puede tener nginx sirviendo contenido estatico y redirigiendo peticiones dinamicas a una aplicacion que consulta PostgreSQL, todo ejecutandose en contenedores Docker, con acceso gestionado a traves de sshd y tareas de mantenimiento ejecutadas por cron. La correcta configuracion de dependencias entre ellos — como que PostgreSQL este activo antes de que la aplicacion arranque — es fundamental para la estabilidad del sistema.

