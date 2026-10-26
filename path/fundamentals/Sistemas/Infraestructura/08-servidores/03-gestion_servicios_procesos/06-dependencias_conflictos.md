# Dependencias y conflictos entre servicios

Un servidor puede ejecutar docenas de servicios simultaneamente, cada uno con sus propios requisitos de red, archivos y recursos. Cuando dos servicios intentan usar el mismo recurso — un puerto TCP, un archivo de bloqueo o un dispositivo — se produce un conflicto que impide que uno de ellos funcione correctamente. Diagnosticar y resolver estos conflictos es una habilidad esencial para cualquier administrador de sistemas.

## Conflictos de puertos

El escenario mas comun de conflicto ocurre cuando dos servicios intentan escuchar en el mismo puerto TCP o UDP. Un puerto solo puede ser utilizado por un proceso a la vez; si un segundo servicio intenta vincularse a un puerto ya en uso, recibira el error `Address already in use` y fallara al iniciar.

El ejemplo clasico es tener dos servidores web intentando usar el puerto 80. Si Apache ya esta escuchando en ese puerto e intentamos iniciar nginx con la misma configuracion, nginx no podra vincularse y terminara con un error. Esto es particularmente peligroso porque el segundo servicio puede fallar silenciosamente o entrar en un ciclo de reinicio continuo si systemd esta configurado con `Restart=on-failure`.

### Diagnosticar conflictos de puertos

La herramienta fundamental para identificar que proceso ocupa un puerto es `ss` (socket statistics), que reemplaza al clasico `netstat`:

```bash
ss -tuln                    # Mostrar todos los puertos en escucha
ss -tuln | grep :80         # Ver que proceso usa el puerto 80
ss -tulnp | grep :80        # Incluir el nombre del proceso (requiere root)
```

El flag `-t` muestra TCP, `-u` muestra UDP, `-l` filtra solo puertos en escucha, `-n` muestra numeros de puerto en lugar de nombres de servicio y `-p` muestra el proceso asociado.

```bash
lsof -i :80                 # Alternativa con lsof
```

`lsof` (list open files) ofrece otra perspectiva, mostrando que proceso tiene abierto el puerto 80. Ambas herramientas son complementarias.

### Resolucion de conflictos

Una vez identificado el conflicto, las opciones dependen del contexto:

1. **Cambiar el puerto de uno de los servicios**: la solucion mas directa. Si nginx debe servir en el puerto 443 y Apache en el 80, simplemente se ajusta la configuracion de uno de ellos para usar un puerto diferente.

2. **Detener el servicio en conflicto**: si no se necesita el segundo servicio, deshabilitarlo con `systemctl disable --now <servicio>` evita que inicie en el futuro.

3. **Usar puertos alternativos**: muchos servicios pueden configurarse para usar puertos no estandarizados. Un segundo servidor web podria usar el puerto 8080, y un proxy inverso redirigir las peticiones.

4. **Escuchar solo en interfaces especificas**: un servicio puede configurarse para escuchar solo en `127.0.0.1` (localhost) o en una IP especifica, permitiendo que otro servicio use el mismo puerto en otra interfaz.

## Conflictos de archivos de bloqueo

Muchos servicios crean archivos de bloqueo (lock files) para evitar que multiples instancias se ejecuten simultaneamente. Si un proceso termina de forma anormal y no elimina su archivo de bloqueo, la siguiente instancia que intente iniciar encontrara el archivo y asumira que otra instancia ya esta activa.

Esto es comun con bases de datos, servidores de correo y daemon de colas de mensajes. El error tipico es algo como `Another instance is already running` o `Lock file exists`.

La solucion es verificar que el proceso realmente no este ejecutando y eliminar manualmente el archivo de bloqueo:

```bash
ps aux | grep <proceso>       # Verificar si realmente hay otra instancia
rm /var/run/<proceso>.lock    # Eliminar el archivo de bloqueo stale
```

> Eliminar un archivo de bloqueo sin verificar primero que el proceso anterior no este activo puede corromper datos si ambas instancias intentan acceder a los mismos recursos concurrentemente.

## Conflictos de dependencias de servicio

systemd gestiona las dependencias entre servicios a traves de sus unit files. Cuando un servicio requiere otro que no esta disponible, puede quedarse en un estado de espera indefinida o fallar al iniciar. Diagnosticar estos problemas implica seguir el grafo de dependencias hasta encontrar el servicio que fallo.

```bash
systemctl list-dependencies nginx.service    # Ver arbol de dependencias
systemctl status postgresql.service          # Verificar estado de la dependencia
```

Un caso tipico es una aplicacion web que necesita PostgreSQL. Si PostgreSQL falla al iniciar (por ejemplo, porque su disco esta lleno o su archivo de configuracion tiene un error), la aplicacion web quedara en estado `failed` o `activating` sin que el error sea evidente a primera vista. Revisar los logs de PostgreSQL con `journalctl -u postgresql` revelara el problema raiz.

## Conflictos de recursos del sistema

Mas alla de puertos y archivos, los servicios pueden competir por recursos del sistema como memoria, CPU y espacio en disco. Un proceso con una fuga de memoria puede consumir toda la RAM disponible, provocando que otros servicios fallen o que el sistema invoque al OOM killer (Out of Memory killer) para terminar procesos arbitrariamente.

```bash
free -h                    # Verificar uso de memoria
df -h                      # Verificar espacio en disco
dmesg | grep -i "out of memory"   # Verificar si OOM killer ha actuado
```

El OOM killer de Linux selecciona procesos para terminar basandose en un score calculado con el consumo de memoria y la prioridad del proceso. Es un mecanismo de emergencia que previene el bloqueo total del sistema, pero puede terminar procesos criticos. Se puede ajustar la prioridad de un proceso frente al OOM killer modificando `/proc/<PID>/oom_score_adj`.

## Estrategias de prevencion

La mejor forma de manejar conflictos es prevenirlos:

- **Asignar puertos unicos por servicio** desde el diseño inicial de la infraestructura. Mantener un registro de que puerto usa cada servicio evita confusiones.
- **Usar nombres descriptivos en los logs** para que los errores incluyan identificadores claros del servicio afectado.
- **Configurar dependencias explicitas** en los unit files de systemd para que los servicios arranquen en el orden correcto.
- **Implementar health checks** que monitoreen periodicamente la salud de cada servicio y alerten antes de que un conflicto cause una caida total.
- **Documentar la configuracion de red** del servidor, incluyendo que servicios usan que puertos, que IPs y que protocolos.

> En entornos de produccion, herramientas como `netdata`, `prometheus` o `monit` pueden monitorear automaticamente el uso de puertos y recursos, alertando proactivamente ante conflictos potenciales antes de que afecten a los usuarios.

