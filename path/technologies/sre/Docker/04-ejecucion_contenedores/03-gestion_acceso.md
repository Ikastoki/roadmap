# Gestion y acceso a contenedores

Una vez que los contenedores estan en ejecucion, es necesario monitorizarlos, controlar su ciclo de vida y acceder a su interior para diagnosticar problemas o ejecutar comandos. Docker proporciona un conjunto de comandos de gestion que permiten listar, detener, reiniciar y eliminar contenedores, asi como inspeccionar su salida y ejecutar procesos dentro de ellos. Estos comandos forman el vocabulario cotidiano de cualquier operador de Docker.

## Listar contenedores: docker ps

`docker ps` muestra los contenedores en ejecucion. Sin argumentos, lista el ID, la imagen, el comando, el tiempo de actividad, los puertos mapeados y el nombre de cada contenedor activo.

```bash
docker ps
# CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
# a1b2c3d4e5f6   nginx   ...       2 min ago Up 2 min 0.0.0.0:8080->80/tcp mi-web
```

La opcion `-a` (all) incluye los contenedores detenidos, mostrando su estado final y el codigo de salida.

```bash
docker ps -a
# Lista tanto contenedores running como stopped
```

Filtros adicionales permiten afinar la busqueda:

```bash
# Contenedores por estado
docker ps -a --filter "status=exited"
docker ps -a --filter "status=running"

# Contenedores por nombre o imagen
docker ps -a --filter "name=mi-web"
docker ps -a --filter "ancestor=nginx"
```

## Detener y reanudar: docker stop/start/restart

`docker stop` envia una senal SIGTERM al proceso PID 1 del contenedor, dandole 10 segundos para realizar una parada ordenada. Si el proceso no termina en ese plazo, Docker envia SIGKILL para forzar la terminacion. `docker start` reactiva un contenedor detenido, reutilizando su capa de escritura y su configuracion original.

```bash
docker stop mi-web
# SIGTERM -> espera 10s -> SIGKILL si es necesario

docker start mi-web
# Reactiva el mismo contenedor con su estado anterior
```

`docker restart` combina stop y start en una sola operacion, con un periodo de gracia configurable.

```bash
docker restart mi-web
# Equivale a stop + start

docker restart --time=30 mi-web
# 30 segundos de gracia antes de SIGKILL
```

## Eliminar contenedores: docker rm

`docker rm` elimina la capa de escritura y la metadata de un contenedor detenido. Un contenedor en ejecucion no puede eliminarse a menos que se use `-f` (force), que primero lo detiene con SIGKILL y luego lo elimina.

```bash
docker rm mi-web                 # Eliminar contenedor detenido
docker rm $(docker ps -aq)       # Eliminar todos los contenedores detenidos
docker rm -f mi-web              # Forzar eliminacion incluso si esta running
```

Para limpiar todos los contenedores detenidos de una sola vez:

```bash
docker container prune
# Confirmacion interactiva; usar -f para saltarla
```

## Acceso interactivo: docker exec

`docker exec` ejecuta un proceso dentro de un contenedor en ejecucion. Es la herramienta principal para inspeccionar el interior de un contenedor sin detenerlo. La combinacion `-it` es necesaria para sesiones interactivas: `-i` mantiene la entrada estandar abierta y `-t` asigna un pseudo-terminal.

```bash
# Acceder al shell de un contenedor en ejecucion
docker exec -it mi-web /bin/bash

# Ejecutar un comando y obtener su salida
docker exec mi-web ls -la /etc/nginx

# Ejecutar un comando en un directorio especifico
docker exec -w /var/log mi-web tail -f access.log
```

`docker exec` no inicia un nuevo contenedor; crea un proceso hijo dentro del namespace del contenedor existente. Esto significa que comparte el mismo sistema de archivos, los mismos procesos y la misma red que el contenedor original.

## Visualizar logs: docker logs

`docker logs` recupera la salida estandar (stdout) y la salida de error (stderr) del proceso PID 1 del contenedor. Es el mecanismo principal para depurar contenedores en produccion, ya que no requiere acceso interactivo.

```bash
# Ver todos los logs
docker logs mi-web

# Seguir los logs en tiempo real (tail -f)
docker logs -f mi-web

# Ultimas N lineas
docker logs --tail 100 mi-web

# Con marcas de tiempo
docker logs -t mi-web
```

Docker captura toda la salida del proceso principal y la almacena en un archivo JSON en el host. Por defecto no hay rotacion de logs, por lo que un contenedor que genera mucha salida puede llenar el disco. Es recomendable configurar el driver de logs con limites.

```bash
# Configurar rotacion de logs al crear el contenedor
docker run -d --name mi-web --log-opt max-size=10m --log-opt max-file=3 nginx
```

## Inspeccion detallada: docker inspect

`docker inspect` devuelve toda la metadata de un contenedor en formato JSON: configuracion, estado, red, montajes, variables de entorno, etc. Es util para obtener informacion estructurada que los comandos de alto nivel no muestran.

```bash
docker inspect mi-web

# Filtrar campos especificos con --format
docker inspect mi-web --format '{{.NetworkSettings.IPAddress}}'
docker inspect mi-web --format '{{.State.Status}}'
docker inspect mi-web --format '{{.Config.Env}}'
```

## Estadisticas en vivo: docker stats

`docker stats` muestra el uso de recursos de los contenedores en ejecucion en tiempo real: CPU, memoria, red y disco. Es util para identificar contenedores que consumen mas recursos de lo esperado.

```bash
docker stats
# Muestra una tabla actualizada cada segundo

docker stats mi-web mi-postgres
# Filtrar por contenedores especificos

docker stats --no-stream
# Una sola captura sin actualizacion continua
```

> La combinacion de `docker logs` para salida de aplicacion y `docker stats` para consumo de recursos cubre la mayoria de las necesidades de monitorizacion en entornos de desarrollo y pruebas. En produccion se usan soluciones externas como Prometheus o Datadog.

