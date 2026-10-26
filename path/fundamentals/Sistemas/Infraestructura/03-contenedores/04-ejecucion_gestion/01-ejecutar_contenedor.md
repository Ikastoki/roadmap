# Ejecutar un contenedor

El comando `docker run` (o `podman run`) es la operacion mas fundamental de la contenerizacion. Crea e inicia un contenedor desde una imagen, aplicando configuraciones de red, almacenamiento, recursos y variables de entorno. Dominar `run` es dominar la contenerizacion.

## Sintaxis basica

```bash
docker run [OPCIONES] IMAGEN [COMANDO]
```

## Opciones comunes

```bash
# Ejecutar en segundo plano (detached)
docker run -d --name mi-web nginx:alpine

# Publicar puertos (-p host:contenedor)
docker run -d -p 8080:80 --name mi-web nginx:alpine
# Acceder: http://localhost:8080

# Variables de entorno (-e)
docker run -d -e POSTGRES_PASSWORD=secret postgres:16-alpine

# Nombre del contenedor (--name)
docker run -d --name mi-base postgres:16-alpine

# Montar volumen (-v)
docker run -d -v mi-datos:/var/lib/postgresql/data postgres:16-alpine

# Montar bind mount (-v ruta_host:ruta_contenedor)
docker run -d -v $(pwd)/html:/usr/share/nginx/html nginx:alpine

# Modo interactivo (-it)
docker run -it alpine sh
# Estas dentro de un shell de Alpine

# Eliminar al detenerse (--rm)
docker run --rm alpine echo "Hola mundo"
# El contenedor se ejecuta, imprime y se elimina automaticamente
```

## Ejemplos practicos

```bash
# Servidor web nginx
docker run -d -p 80:80 --name web nginx:alpine

# Base de datos PostgreSQL con datos persistentes
docker run -d \
  --name mi-db \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mibasededatos \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16-alpine

# Redis con datos persistentes
docker run -d \
  --name mi-cache \
  -v redis-data:/data \
  -p 6379:6379 \
  redis:7-alpine redis-server --appendonly yes

# Aplicacion Node.js con variables de entorno
docker run -d \
  --name mi-app \
  -e NODE_ENV=production \
  -e DB_HOST=mi-db \
  -e DB_PORT=5432 \
  -p 3000:3000 \
  --link mi-db \
  mi-app:1.0

# Ejecutar comando especifico en lugar del CMD por defecto
docker run --rm postgres:16-alpine psql --version
# Imprime la version de psql y se detiene
```

## Ejecucion avanzada

```bash
# Limitar recursos
docker run -d \
  --memory 512m \
  --cpus 1.5 \
  --pids-limit 100 \
  mi-app

# Red especifica
docker run -d --network mi-red nginx:alpine

# Montar volumen de solo lectura
docker run -d -v $(pwd)/config:/etc/nginx/nginx.conf:ro nginx:alpine

# Ejecutar con usuario especifico
docker run -d -u 1000:1000 nginx:alpine

# Ejecutar con hostname especifico
docker run -d --hostname mi-servidor nginx:alpine

# Ejecutar con restricciones de seguridad
docker run -d --read-only --tmpfs /tmp nginx:alpine
# Contenedor de solo lectura, con /tmp writable en memoria
```

## Diferencia entre CMD y ENTRYPOINT

```bash
# CMD: se puede sobreescribir con argumentos
docker run --rm node:20 node -v
# node -v sobreescribe el CMD por defecto

# ENTRYPOINT: no se puede sobreescribir (siempre se ejecuta)
docker run --rm node:20 -v
# Esto fallaria porque -v no es un comando valido
```

> `docker run` es la puerta de entrada a los contenedores. Cada opcion (-d, -p, -v, -e, --name) controla un aspecto del comportamiento del contenedor. La combinacion correcta de estas opciones crea un contenedor que se ejecuta de forma segura, eficiente y predecible.

