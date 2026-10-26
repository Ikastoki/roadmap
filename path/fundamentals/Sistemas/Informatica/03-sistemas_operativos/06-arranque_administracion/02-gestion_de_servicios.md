# Gestion de servicios

La gestion de servicios se refiere al control de los demonios (procesos en segundo plano) que proporcionan las funcionalidades del sistema: servidores web, bases de datos, redes, impresion, etc. En Linux moderno, `systemd` es el sistema de gestion de servicios que ha reemplazado al init tradicional (SysVinit) en la mayoria de las distribuciones.

systemd no solo gestiona servicios: tambien gestiona dispositivos, montajes, timers, sockets y otros recursos del sistema a traves de unidades (*units*) con una interfaz uniforme.

## Systemd: unidades y targets

Cada elemento gestionado por systemd es una **unidad** (*unit*) definida en un archivo de configuracion. Los tipos de unidades mas comunes son:

- **.service:** un servicio o demonio (nginx, sshd, mysql).
- **.socket:** un socket de escucha para activacion por demanda.
- **.mount:** un punto de montaje del sistema de archivos.
- **.timer:** un temporizador que activa unidades periodicamente (similar a cron).
- **.target:** un grupo de unidades que representan un nivel de funcionalidad (multi-user, graphical).

Los archivos de servicio se almacenan en `/lib/systemd/system/` (paquetes) y `/etc/systemd/system/` (configuracion local).

```bash
# Archivo de servicio personalizado
# /etc/systemd/system/mi-app.service
[Unit]
Description=Mi aplicacion web
After=network.target
Requires=mysql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/mi-app
ExecStart=/opt/mi-app/server
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Comandos basicos de systemctl

`systemctl` es la herramienta principal para interactuar con systemd:

```bash
# Gestion de servicios
sudo systemctl start nginx      # Iniciar
sudo systemctl stop nginx       # Detener
sudo systemctl restart nginx    # Reiniciar (detener + iniciar)
sudo systemctl reload nginx     # Recargar configuracion sin reiniciar
sudo systemctl status nginx     # Ver estado y ultimas lineas de log

# Habilitar/deshabilitar en el arranque
sudo systemctl enable nginx     # Habilitar arranque automatico
sudo systemctl disable nginx    # Deshabilitar arranque automatico
sudo systemctl enable --now nginx  # Habilitar + iniciar inmediatamente

# Listar unidades
systemctl list-units --type=service                   # Servicios activos
systemctl list-units --type=service --state=running   # Servicios en ejecucion
systemctl list-unit-files --type=service              # Todos los servicios instalados
```

## Dependencias entre servicios

systemd gestiona las dependencias entre servicios automaticamente. Los directivas clave en los archivos de servicio son:

- **After=network.target:** el servicio se arranca despues de que la red este disponible.
- **Requires=mysql.service:** el servicio necesita mysql; si mysql falla, este servicio tambien se detiene.
- **Wants=redis.service:** el servicio desea redis; si redis falla, este servicio continua.
- **Conflicts=iptables.service:** el servicio no puede ejecutarse simultaneamente con iptables.

```bash
# Ver dependencias de un servicio
systemctl list-dependencies nginx.service
systemd-analyze critical-chain nginx.service
```

## Modos de ejecucion (targets)

Los *targets* son el equivalente moderno de los runlevels de SysVinit:

- **multi-user.target:** modo texto con red (equivalente a runlevel 3). Servidores.
- **graphical.target:** modo grafico con escritorio (equivalente a runlevel 5). Escritorios.
- **rescue.target:** modo monousuario para recuperacion (equivalente a runlevel 1).
- **emergency.target:** modo de emergencia con sistema de archivos raiz montado como solo lectura.

```bash
# Cambiar de target
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target

# Establecer target por defecto
sudo systemctl set-default multi-user.target
sudo systemctl get-default
```

## Gestion de servicios con systemd

systemd permite la activacion por demanda de servicios a traves de sockets: el servicio no se arranca hasta que alguien intenta conectarse al socket. Esto reduce el tiempo de arranque y el consumo de memoria:

```bash
# Activacion por socket
# /etc/systemd/system/mi-servicio.socket
[Unit]
Description=Socket de mi servicio

[Socket]
ListenStream=8080

[Install]
WantedBy=sockets.target
```

Los *timers* de systemd son el reemplazo moderno de cron para tareas periodicas:

```bash
# /etc/systemd/system/limpieza.timer
[Unit]
Description=Timer de limpieza diaria

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

> systemd ha transformado la gestion de servicios en Linux, proporcionando una interfaz unificada y dependencias declarativas. Aunque su adopcion fue controvertida (criticada por violar el principio de hacer una sola cosa), su eficiencia y funcionalidad lo han convertido en el estandar de facto en todas las distribuciones principales.

