# Comandos esenciales de Compose

Los comandos de Docker Compose gestionan el ciclo de vida completo de un stack multi-contenedor: desde la creacion inicial hasta la eliminacion, pasando por diagnostico, escalado y mantenimiento.

## Ciclo de vida

```bash
# Crear e iniciar todos los servicios
docker compose up -d

# Ver servicios activos
docker compose ps
# NAME         IMAGE          STATUS       PORTS
# web          nginx:alpine   Up 2 min    0.0.0.0:8080->80/tcp
# api          node:20        Up 2 min
# db           postgres:16    Up 2 min    5432/tcp

# Detener servicios (conserva contenedores)
docker compose stop

# Iniciar servicios detenidos
docker compose start

# Detener y eliminar contenedores, redes
docker compose down

# Detener y eliminar TODO (incluidos volumenes anonimos)
docker compose down -v
```

## Logs y diagnostico

```bash
# Ver logs de todos los servicios
docker compose logs

# Seguir logs en tiempo real
docker compose logs -f

# Logs de un servicio especifico
docker compose logs -f api

# Ultimas 100 lineas
docker compose logs --tail 100 api

# Logs con timestamps
docker compose logs -t api
```

## Ejecucion de comandos

```bash
# Ejecutar comando en un servicio en ejecucion
docker compose exec api node -e "console.log('hello')"

# Shell interactivo
docker compose exec db psql -U admin -d miapp

# Ejecutar como usuario especifico
docker compose exec --user root api bash

# Ejecutar en servicio que no esta corriendo
docker compose run --rm api npm test
# --rm: eliminar contenedor despues de ejecutar
```

## Reconstruccion

```bash
# Reconstruir imagenes y recrear contenedores
docker compose up -d --build

# Reconstruir un servicio especifico
docker compose up -d --build api

# Forzar reconstruccion sin caché
docker compose build --no-cache api

# Pull de imagenes actualizadas
docker compose pull
docker compose up -d
```

## Escalado

```bash
# Escalar un servicio a multiples instancias
docker compose up -d --scale api=3
# Crea 3 contenedores del servicio api

# Ver instancias
docker compose ps
# NAME         IMAGE     STATUS
# api-1        node:20   Up
# api-2        node:20   Up
# api-3        node:20   Up
```

## Limpieza

```bash
# Eliminar contenedores detenidos
docker compose rm

# Eliminar imagenes sin usar del proyecto
docker image prune -f

# Limpiar todo lo relacionado con el proyecto
docker compose down -v --rmi all
```

> Dominar los comandos de Compose es dominar el ciclo de vida de un stack multi-contenedor. La combinacion de `up -d`, `logs -f`, `exec` y `down` cubre el 90% de las operaciones diarias.

