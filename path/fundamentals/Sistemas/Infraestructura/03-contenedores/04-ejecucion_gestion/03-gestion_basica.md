# Gestion basica de contenedores

La gestion basica de contenedores incluye listar, detener, iniciar, eliminar y reiniciar contenedores. Estas operaciones son las que se realizan con mayor frecuencia en el dia a dia y son el fundamento para administrar cualquier entorno de contenedores.

## Listar contenedores

```bash
# Contenedores en ejecucion
docker ps
# CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                  NAMES
# a1b2c3d4e5f6   nginx:alpine   "/docker-entrypoint.…"   5 minutes ago    Up 5 minutes    0.0.0.0:80->80/tcp     mi-web
# b2c3d4e5f6a1   postgres:16    "docker-entrypoint.s…"   10 minutes ago   Up 10 minutes   0.0.0.0:5432->5432/tcp mi-db

# Todos los contenedores (incluyendo detenidos)
docker ps -a
# CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS                     NAMES
# a1b2c3d4e5f6   nginx:alpine   "/docker-entrypoint.…"   5 minutes ago    Up 5 minutes               mi-web
# c3d4e5f6a1b2   alpine         "echo Hola"              1 hour ago       Exited (0) 1 hour ago      task-1

# Listar solo IDs
docker ps -q
# a1b2c3d4e5f6
# b2c3d4e5f6a1

# Filtrar por estado
docker ps --filter "status=running"
docker ps --filter "status=exited"
docker ps --filter "ancestor=nginx"
```

## Detener contenedores

```bash
# Detener un contenedor (SIGTERM → SIGKILL despues de 10s)
docker stop mi-web

# Detener multiples contenedores
docker stop mi-web mi-db mi-cache

# Detener todos los contenedores en ejecucion
docker stop $(docker ps -q)

# Detener con timeout personalizado
docker stop -t 30 mi-web    # Esperar 30 segundos
docker stop -t 0 mi-web     # SIGKILL inmediato
```

## Iniciar contenedores

```bash
# Iniciar un contenedor detenido
docker start mi-web

# Iniciar multiples
docker start mi-web mi-db

# Iniciar todos los detenidos
docker start $(docker ps -a -q --filter "status=exited")
```

## Reiniciar contenedores

```bash
# Reiniciar (stop + start)
docker restart mi-web

# Reiniciar con timeout
docker restart -t 5 mi-web
```

## Eliminar contenedores

```bash
# Eliminar contenedor detenido
docker rm mi-web

# Forzar eliminacion (matar y eliminar)
docker rm -f mi-web

# Eliminar multiples
docker rm mi-web mi-db mi-cache

# Eliminar todos los contenedores detenidos
docker rm $(docker ps -a -q --filter "status=exited")

# Eliminar todos los contenedores (con precaucion)
docker rm -f $(docker ps -aq)

# Limpiar contenedores detenidos (similar a docker system prune)
docker container prune
# Delete all stopped containers? [y/N]
# Total reclaimed space: 150MB
```

## Inspeccionar contenedores

```bash
# Ver metadatos completos
docker inspect mi-web

# Ver IP del contenedor
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mi-web
# 172.17.0.2

# Ver puertos mapeados
docker inspect --format='{{json .NetworkSettings.Ports}}' mi-web
# {"80/tcp":[{"HostIp":"0.0.0.0","HostPort":"80"}]}

# Ver volumenes montados
docker inspect --format='{{json .Mounts}}' mi-web
# [{"Type":"volume","Name":"mi-datos","Source":"/var/lib/docker/volumes/mi-datos/_data"}]
```

## Logs

```bash
# Ver logs de un contenedor
docker logs mi-web
# 172.17.0.1 - - [15/Jan/2024:10:30:00 +0000] "GET / HTTP/1.1" 200

# Seguir logs en tiempo real
docker logs -f mi-web

# Ver ultimas 10 lineas
docker logs --tail 10 mi-web

# Ver logs con timestamps
docker logs -t mi-web
```

> La gestion basica de contenedores es intuitiva y consistente entre motores. Listar, detener, iniciar y eliminar son operaciones que se ejecutan en segundos. La clave esta en mantener los contenedores ordenados con nombres descriptivos y en limpiar los contenedores detenidos regularmente.

