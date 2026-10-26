# Acceso y diagnostico de contenedores

Cuando un contenedor no se comporta como se espera, el diagnostico requiere acceder a su interior, revisar logs, verificar la red y comprobar los recursos. Docker y Podman proporcionan herramientas para inspeccionar, ejecutar comandos y monitorear contenedores sin necesidad de reinstalar el SO.

## Exec: ejecutar comandos dentro del contenedor

```bash
# Abrir shell interactivo
docker exec -it mi-web sh
# Dentro del contenedor, puedes explorar el SO, revisar archivos, etc.

# Ejecutar un comando especifico
docker exec mi-web cat /etc/nginx/nginx.conf
docker exec mi-web ps aux
docker exec mi-web netstat -tlnp

# Ejecutar como usuario especifico
docker exec -u 1000 mi-web whoami

# Ejecutar con variables de entorno
docker exec -e MI_VAR=valor mi-web env
```

## Logs: ver la salida del contenedor

```bash
# Ver todos los logs
docker logs mi-web

# Seguir logs en tiempo real (como tail -f)
docker logs -f mi-web

# Ver ultimas N lineas
docker logs --tail 50 mi-web

# Ver logs con timestamps
docker logs -t mi-web

# Ver logs desde un momento especifico
docker logs --since 2024-01-15T10:00:00 mi-web
docker logs --since 30m mi-web    # Ultimos 30 minutos

# Ver logs hasta un momento especifico
docker logs --until 2024-01-15T11:00:00 mi-web
```

## Top: ver procesos dentro del contenedor

```bash
# Ver procesos activos (similar a ps aux)
docker top mi-web
# UID   PID   PPID  C   STIME  TTY   TIME      CMD
# root  1234  1230  0   10:30  ?     00:00:00  nginx: master process nginx -g daemon off;
# root  1235  1234  0   10:30  ?     00:00:00  nginx: worker process
```

## Stats: monitoreo en tiempo real

```bash
# Estadisticas de uso de recursos de todos los contenedores
docker stats
# CONTAINER   CPU %   MEM USAGE / LIMIT     MEM %   NET I/O           BLOCK I/O
# mi-web      0.50%   15.2MiB / 512MiB      2.97%   1.2kB / 2.5kB     0B / 0B
# mi-db       2.30%   45.8MiB / 1GiB        4.47%   5.2kB / 3.1kB     12MB / 8MB

# Estadisticas de un contenedor especifico
docker stats mi-web --no-stream
```

## Inspeccion detallada

```bash
# Ver configuracion completa
docker inspect mi-web

# Ver IP y red
docker inspect --format='{{.NetworkSettings.IPAddress}}' mi-web

# Ver puertos mapeados
docker inspect --format='{{json .NetworkSettings.Ports}}' mi-web

# Ver volumenes
docker inspect --format='{{json .Mounts}}' mi-web

# Ver variables de entorno
docker inspect --format='{{json .Config.Env}}' mi-web

# Ver estado
docker inspect --format='{{.State.Status}}' mi-web
docker inspect --format='{{.State.StartedAt}}' mi-web
docker inspect --format='{{.State.Health}}' mi-web
```

## Diagnostico de red

```bash
# Ver redes del contenedor
docker inspect --format='{{json .NetworkSettings.Networks}}' mi-web

# Probar conectividad desde el contenedor
docker exec mi-web ping -c 3 google.com

# Ver puertos abiertos dentro del contenedor
docker exec mi-web netstat -tlnp
# o:
docker exec mi-web ss -tlnp
```

## Copiar archivos

```bash
# Copiar archivo del host al contenedor
docker cp config.json mi-web:/app/config.json

# Copiar archivo del contenedor al host
docker cp mi-web:/app/logs/app.log ./logs/

# Copiar directorio
docker cp mi-web:/app/data ./data-backup
```

> El diagnostico de contenedores es similar al diagnostico de procesos normales, pero con herramientas especificas del motor de contenedores. exec, logs, stats y inspect son las cuatro herramientas fundamentales para entender que esta ocurriendo dentro de un contenedor.

