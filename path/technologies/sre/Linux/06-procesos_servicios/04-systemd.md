# Servicios con systemd

systemd es el sistema de inicio (init) y gestor de servicios adoptado practicamente por todas las distribuciones Linux modernas (Debian, Ubuntu, Fedora, Arch, CentOS, RHEL). Reemplaza al antiguo System V init con un enfoque basado en unidades, paralelizacion del arranque y activacion bajo demanda. A diferencia de SysV, donde los scripts de inicio eran secuenciales y escritos en shell, systemd define servicios, sockets, temporizadores y puntos de montaje como archivos de configuracion declarativos.

## Unidades de systemd

Una unidad (unit) es un recurso gestionado por systemd. Cada unidad se define en un archivo `.service`, `.socket`, `.timer`, `.mount`, `.path`, `.target` o `.device`. Los archivos de unidad residen en `/lib/systemd/system/` (provistos por paquetes), `/etc/systemd/system/` (personalizacion del administrador) y `/run/systemd/system/` (unidades transitorias en memoria). La precedencia es: `/etc` sobreescribe `/lib`, y `/run` tiene la maxima prioridad.

```bash
# Listar todas las unidades disponibles
systemctl list-unit-files

# Listar unidades de servicio activas
systemctl list-units --type=service

# Listar unidades fallidas
systemctl list-units --failed
```

Estructura tipica de una unidad de servicio (`/etc/systemd/system/mi-servicio.service`):

```ini
[Unit]
Description=Mi servicio personalizado
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/mi-app
ExecStart=/usr/bin/python3 /opt/mi-app/server.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

La directiva `Type` define como systemd determina si el servicio se ha iniciado correctamente:
- `simple`: el proceso principal es el servicio (no notifica a systemd)
- `forking`: el proceso principal hace fork y systemd espera al hijo
- `oneshot`: el proceso termina pero se considera activo (tipico para tareas unicas)
- `notify`: el proceso envia un mensaje a systemd via `sd_notify()` cuando esta listo

> La directiva `Restart` evita escribir bucles de watchdog manuales. `on-failure` reinicia el servicio solo si termina con codigo de salida distinto de cero o recibe una senal fatal. `always` lo reinicia incluso si termina normalmente.

## systemctl: control de servicios

`systemctl` es la herramienta principal para gestionar servicios y unidades systemd. Su uso es intuitivo una vez se comprenden los verbos basicos.

```bash
# Iniciar un servicio
sudo systemctl start nginx

# Detener un servicio
sudo systemctl stop nginx

# Reiniciar un servicio
sudo systemctl restart nginx

# Recargar configuracion sin detener el servicio (si soportado)
sudo systemctl reload nginx

# Habilitar inicio automatico al arrancar
sudo systemctl enable nginx

# Deshabilitar inicio automatico
sudo systemctl disable nginx

# Ver estado detallado del servicio
sudo systemctl status nginx

# Verificar si un servicio esta activo
systemctl is-active nginx

# Verificar si un servicio esta habilitado
systemctl is-enabled nginx
```

La salida de `systemctl status` incluye: el archivo de unidad cargado, el estado activo (active/inactive/failed), el PID principal, el tiempo de ejecucion, el uso de memoria, las tareas del cgroup y las ultimas lineas del log del servicio.

```bash
# Recargar la configuracion de systemd (tras crear/modificar unidades)
sudo systemctl daemon-reload

# Enmascarar un servicio (imposible de iniciar incluso manualmente)
sudo systemctl mask servicio

# Descubrir un servicio
sudo systemctl unmask servicio

# Ver dependencias de una unidad
systemctl list-dependencies nginx
```

> `systemctl daemon-reload` es obligatorio tras crear o modificar un archivo de unidad. No reinicia los servicios en ejecucion; solo recarga la configuracion para que systemd reconozca los cambios en la proxima operacion.

## journalctl: visualizacion de logs

systemd unifica la gestion de logs a traves de journald, su sistema de registro binario. `journalctl` permite consultar estos logs con filtros potentes que van mucho mas alla de leer archivos de texto en `/var/log/`.

```bash
# Ver todos los logs del sistema (requiere permisos)
sudo journalctl

# Seguir los logs en tiempo real (como tail -f)
sudo journalctl -f

# Ver logs de un servicio especifico
sudo journalctl -u nginx

# Logs desde el ultimo arranque
sudo journalctl -b

# Logs de un arranque anterior (indice negativo)
sudo journalctl -b -1

# Logs de las ultimas N lineas
sudo journalctl -n 50

# Logs desde una marca temporal
sudo journalctl --since "2024-01-01 00:00:00" --until "2024-01-02 00:00:00"

# Logs desde hace 1 hora
sudo journalctl --since "1 hour ago"
```

journalctl formatea la salida con colores y paginacion automatica. Para extraer datos en crudo:

```bash
# Salida sin paginacion
sudo journalctl --no-pager

# Formato JSON (util para ingestion en herramientas de monitoring)
sudo journalctl -u nginx -o json-pretty

# Solo mensajes de error
sudo journalctl -p err

# Prioridades disponibles: emerg, alert, crit, err, warning, notice, info, debug
```

> journalctl almacena los logs en formato binario en `/var/log/journal/`. Son resistentes a la corruption y ocupan menos espacio que los logs de texto plano. Para persistencia entre reinicios, asegurate de que `/var/log/journal/` existe (`sudo mkdir -p /var/log/journal`).

## Temporizadores y otros tipos de unidad

systemd reemplaza cron con temporizadores (`.timer`). Un temporizador activa un servicio asociado en momentos definidos.

```bash
# Listar temporizadores activos
systemctl list-timers

# Crear un temporizador (archivo /etc/systemd/system/mi-tarea.timer)
[Unit]
Description=Ejecutar mi tarea cada hora

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
```

Otros tipos de unidad amplian el alcance de systemd:
- `.socket`: activa un servicio cuando llega trafico a un socket de red o Unix
- `.path`: activa un servicio cuando un archivo o directorio cambia
- `.mount`: gestiona puntos de montaje como unidades
- `.target`: agrupa unidades en estados del sistema (multi-user.target, graphical.target)

```bash
# Ver el objetivo de arranque por defecto
systemctl get-default

# Cambiar al modo multiusuario (sin interfaz grafica)
sudo systemctl set-default multi-user.target
```

systemd reduce el tiempo de arranque al paralelizar la activacion de servicios y retrasar la inicializacion de aquellos que no son necesarios inmediatamente. Un servicio solo se inicia cuando otro servicio o un socket lo requiere, eliminando el arranque secuencial de SysV init.

