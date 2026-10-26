# Logs del sistema

El registro de eventos es la columna vertebral del diagnostico en servidores. Cuando un servicio falla, un usuario no puede autenticarse o el rendimiento se degrada, la primera accion es revisar los logs. Linux dispone de dos sistemas de registro que coexisten: los archivos de log tradicionales en `/var/log/` y journald, el subsistema integrado en systemd.

## journald: el registro centralizado de systemd

journald es el daemon de logging que acompana a systemd. Recopila logs de todas las fuentes del sistema — kernels, demonios, servicios systemd, syslog remoto — y los almacena en un formato binario indexado. Esto permite busquedas rapidas y eficientes que serian costosas con archivos de texto plano.

### journalctl: consultar los logs

`journalctl` es la herramienta de linea de comandos para interactuar con el registro de journald. Su uso mas basico muestra todos los mensajes del sistema:

```bash
journalctl                    # Todos los logs desde el ultimo arranque
journalctl -b                 # Logs del arranque actual
journalctl -b -1              # Logs del arranque anterior
```

La verdadero poder de journalctl radica en sus filtros. Puedes combinar multiples criterios para localizar rapidamente los eventos relevantes:

```bash
journalctl -u nginx.service              # Logs de un servicio especifico
journalctl -u nginx.service -f           # Seguimiento en tiempo real (como tail -f)
journalctl -p err                        # Solo mensajes de nivel error o superior
journalctl --since "2025-01-01" --until "2025-01-02"  # Rango de fechas
journalctl -u nginx --since "1 hour ago"  # Logs recientes de un servicio
journalctl _PID=1234                     # Logs de un PID especifico
```

Los niveles de severidad siguen el estandar syslog: `emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info` y `debug`. Usar `-p err` o `-p warning` permite filtrar rapidamente los problemas importantes entre el ruido de mensajes informativos.

### Gestion del almacenamiento de journals

Los archivos de journal se almacenan en `/var/log/journal/` en formato binario. Por defecto, systemd puede configurar un tamaño maximo de almacenamiento en `/etc/systemd/journald.conf` con la directiva `SystemMaxUse`. Cuando se supera el limite, los registros mas antiguos se eliminan automaticamente.

Para consultar el espacio que ocupan los logs:

```bash
journalctl --disk-usage
```

Y para vaciar el journal completamente:

```bash
journalctl --rotate
journalctl --vacuum-size=500M
```

> Si `/var/log/journal/` no existe o esta configurado como volatile (`Storage=none`), los logs se pierden al reiniciar el sistema. En servidores de produccion, siempre se debe configurar almacenamiento persistente.

## Archivos de log tradicionales en /var/log/

Antes de journald, Linux utilizaba archivos de texto plano en `/var/log/` para el registro de eventos. Aunque systemd ha reducido su dependencia, estos archivos siguen siendo importantes porque muchos servicios los escriben directamente.

### syslog y messages

`/var/log/syslog` (en Debian/Ubuntu) o `/var/log/messages` (en RHEL/CentOS) es el archivo general del sistema. Contiene mensajes de la mayoria de servicios que no tienen su propio archivo de log. Historicamente, el daemon `syslog` o `rsyslog` era el encargado de recopilar y clasificar estos mensajes en archivos segun su facility y prioridad.

### auth.log y secure

`/var/log/auth.log` (Debian/Ubuntu) o `/var/log/secure` (RHEL/CentOS) registra todos los eventos de autenticacion: inicios de sesion exitosos y fallidos, cambios de usuario, uso de sudo y conexiones SSH. Es el primer archivo que se revisa ante sospechas de acceso no autorizado.

```bash
grep "Failed password" /var/log/auth.log | tail -20    # Ultimos 20 intentos fallidos
grep "Accepted publickey" /var/log/auth.log             # Inicios de sesion exitosos con clave
```

### Log de nginx

nginx escribe logs de acceso y error en archivos configurados en `nginx.conf`, tipicamente en `/var/log/nginx/access.log` y `/var/log/nginx/error.log`. El log de acceso registra cada peticion HTTP recibida con su codigo de respuesta, tiempo de procesamiento y tamanyo, lo que permite analizar trafico y detectar problemas de rendimiento.

### Rotacion de logs: logrotate

Los archivos de log crecen indefinidamente si no se gestionan. `logrotate` es la herramienta clasica que rota, comprime y elimina logs antiguos. Se configura en `/etc/logrotate.conf` y archivos individuales en `/etc/logrotate.d/`.

Una configuracion tipica para nginx:

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

Esto define que los logs se rotan diariamente, se conservan 14 dias comprimidos, y se envia `SIGUSR1` a nginx para que reabra los archivos de log despues de la rotacion.

## journald vs archivos tradicionales

En un sistema moderno con systemd, ambos mecanismos coexisten. journald ofrece busquedas indexadas, metadatos enriquecidos (PID, usuario, unidad systemd) y filtrado por campo. Los archivos de texto tradicionales son mas simples, universales y fáciles de procesar con herramientas como `grep`, `awk` o `sed`.

Muchos servicios escriben simultaneamente en ambos sistemas. Por ejemplo, rsyslog puede leer los mensajes de journald y escribirlos en archivos tradicionales, permitiendo que herramientas externas de analisis de logs (ELK, Splunk) consuman los datos de forma sencilla.

```bash
systemctl status rsyslog    # Verificar que rsyslog esta activo
```

> En entornos de produccion con multiples servidores, la combinacion de journald con un sistema centralizado de gestion de logs (como el stack ELK o Loki) es la practica recomendada para correlacionar eventos entre diferentes maquinas.

