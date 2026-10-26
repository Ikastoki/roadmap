# Tareas programadas

La automatizacion de tareas periodicas es un pilar de la administracion de servidores. Respaldos nocturnos, rotacion de logs, limpieza de archivos temporales, actualizaciones de indices y monitoreo de salud del sistema son operaciones que deben ejecutarse sin intervencion humana. Linux ofrece dos mecanismos principales para esto: cron, el clasico, y systemd timers, la alternativa moderna.

## cron: el demonio clasico

cron es uno de los servicios mas antiguos de Unix y sigue presente en praticamente todas las distribuciones Linux. Su funcion es simple: comparar la hora actual contra las reglas definidas en archivos de planificacion y ejecutar los comandos correspondientes.

### Estructura del crontab

Cada usuario puede tener su propio archivo de planificacion (crontab). El formato de cada linea sigue el patron:

```
minuto hora dia mes dia_semana comando
```

Los campos aceptan valores numericos, rangos (`1-5`), listas (`1,3,5`) y asteriscos (`*` para "cada vez"). Algunos ejemplos:

```
# Ejecutar a las 3:00 AM todos los dias
0 3 * * * /usr/local/bin/backup.sh

# Ejecutar cada 15 minutos
*/15 * * * * /usr/local/bin/check_health.sh

# Ejecutar lunes a viernes a las 8:00 AM
0 8 * * 1-5 /usr/local/bin/report.sh

# Ejecutar el primer dia de cada mes a medianoche
0 0 1 * * /usr/local/bin/monthly_cleanup.sh
```

El crontab del root se edita con `crontab -e` y se puede inspeccionar con `crontab -l`. Los cron jobs del sistema se encuentran en `/etc/crontab` y en los directorios `/etc/cron.d/`, `/etc/cron.daily/`, `/etc/cron.hourly/`, `/etc/cron.weekly/` y `/etc/cron.monthly/`.

### Variables de entorno en cron

El entorno de ejecucion de cron es muy limitado comparado con una sesion interactiva. No carga las variables de PATH habituales, lo que es una fuente comun de fallos. Es recomendable definir PATH explicitamente al inicio del crontab:

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Ademas, cron redirige la salida estandar y de error por correo al usuario. Si no hay un sistema de correo configurado, estas salidas se pierden silenciosamente. Por eso es buena practica redirigir la salida manualmente:

```
0 3 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

## systemd timers: la alternativa moderna

systemd timers ofrecen las mismas capacidades que cron pero con ventajas significativas en integracion, gestion de errores y flexibilidad de planificacion.

### Concepto de temporizador

Un systemd timer funciona en conjunto con un servicio. El temporizador define **cuando** ejecutar y el servicio define **que** ejecutar. Se crean dos archivos: uno `.timer` y uno `.service` con el mismo nombre base.

Archivo de servicio (`/etc/systemd/system/limpieza.service`):

```ini
[Unit]
Description=Limpieza de archivos temporales

[Service]
Type=oneshot
ExecStart=/usr/local/bin/limpieza.sh
```

Archivo de temporizador (`/etc/systemd/system/limpieza.timer`):

```ini
[Unit]
Description=Ejecutar limpieza diariamente

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Despues de crear ambos archivos, se habilita y arranca el temporizador:

```bash
systemctl daemon-reload
systemctl enable --now limpieza.timer
```

### Expresiones de calendario

systemd timers ofrecen un lenguaje mucho mas expresivo que cron para definir horarios:

- `OnCalendar=daily`: una vez al dia a medianoche.
- `OnCalendar=hourly`: cada hora.
- `OnCalendar=*-*-* 03:00:00`: todos los dias a las 3:00 AM.
- `OnCalendar=Mon *-*-* 08:00:00`: todos los lunes a las 8:00 AM.
- `OnBootSec=5min`: 5 minutos despues del arranque del sistema.
- `OnUnitActiveSec=30min`: 30 minutos despues de la ultima ejecucion.

### Ventajas sobre cron

La ventaja mas importante es `Persistent=true`. Si el servidor estaba apagado cuando debia ejecutarse una tarea, systemd la ejecutara automaticamente al arrancar. cron no tiene esta capacidad por defecto.

Ademas, cada ejecucion de un timer genera un registro en el journal del sistema. Esto permite inspeccionar el historial completo de ejecuciones con `journalctl -u limpieza.service`, incluyendo si la tarea fallo y por que. Con cron, el registro de ejecuciones requiere configuracion adicional.

Los timers tambien integran plenamente con las dependencias de systemd. Si defines `Requires=network.target` en el servicio, systemd no ejecutara la tarea si la red no esta disponible, algo que cron no puede hacer nativamente.

## Cron vs systemd timers

| Aspecto | cron | systemd timers |
|---|---|---|
| Integracion con logs | Manual, requiere redirigir salida | Automatica via journald |
| Ejecucion de tareas perdidas | No (si el servidor estaba apagado) | Si, con `Persistent=true` |
| Dependencias de servicios | No soportadas | Completo soporte via systemd |
| Planificacion | Basicamente periodos fijos | Expresiones de calendario flexibles |
| Uso de recursos | Ligero, daemon siempre activo | Mas overhead pero insignificante |
| Disponibilidad | Universal en Linux | Requiere systemd |

En la practica, para tareas simples en servidores que no usan systemd intensivamente, cron sigue siendo una opcion solida y bien understood. Para entornos modernos donde ya se usa systemd como sistema init, los timers son preferibles por su integracion y capacidades superiores.

> Si migras de cron a systemd timers, asegurate de que `systemctl list-timers --all` muestre tus temporizadores activos antes de eliminar los crontabs correspondientes.

