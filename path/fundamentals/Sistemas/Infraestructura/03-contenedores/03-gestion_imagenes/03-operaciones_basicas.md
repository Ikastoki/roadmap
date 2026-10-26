# Operaciones basicas con imagenes

Las operaciones basicas de imagenes son las primeras herramientas que todo usuario de contenedores debe dominar: pull (descargar), list (ver), remove (eliminar) y tag (etiquetar). Estas operaciones son el fundamento para trabajar con cualquier motor de contenedores.

## Pull: descargar imagenes

```bash
# Descargar imagen oficial de nginx
docker pull nginx
# Using default tag: latest
# latest: Pulling from library/nginx
# a2abf6c4d29d: Pull complete
# Digest: sha256:...
# Status: Downloaded newer image for nginx:latest

# Descargar version especifica
docker pull nginx:1.25-alpine

# Descargar de un registro privado
docker pull registry.empresa.com/mi-app:1.0

# Descargar de GitHub Container Registry
docker pull ghcr.io/usuario/proyecto:1.0
```

## Images: listar imagenes locales

```bash
# Listar todas las imagenes
docker images
# REPOSITORY   TAG          IMAGE ID       CREATED        SIZE
# nginx        1.25-alpine  a1b2c3d4e5f6   2 weeks ago    42.5MB
# nginx        latest       b2c3d4e5f6a1   2 weeks ago    187MB
# postgres     16-alpine    c3d4e5f6a1b2   3 weeks ago    238MB
# alpine       latest       d4e5f6a1b2c3   1 month ago    7.74MB

# Listar solo IDs
docker images -q
# a1b2c3d4e5f6
# b2c3d4e5f6a1

# Filtrar por repository
docker images nginx
# REPOSITORY   TAG          IMAGE ID       CREATED        SIZE
# nginx        1.25-alpine  a1b2c3d4e5f6   2 weeks ago    42.5MB
# nginx        latest       b2c3d4e5f6a1   2 weeks ago    187MB

# Ver tamano total
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          4         2         475.2MB   237.6MB (50%)
# Containers      2         1         0B        0B
# Local Volumes   3         1         150MB     100MB (66%)
```

## RMI: eliminar imagenes

```bash
# Eliminar imagen por tag
docker rmi nginx:latest
# Untagged: nginx:latest
# Deleted: sha256:b2c3d4e5f6a1...

# Eliminar imagen por ID
docker rmi a1b2c3d4e5f6

# Eliminar todas las imagenes (con precaucion)
docker rmi $(docker images -q)

# Eliminar imagenes colgadas (sin tag)
docker image prune
# Total reclaimed space: 125.4MB

# Eliminar todas las imagenes no usadas
docker image prune -a
# WARNING! This will remove all images without at least one container associated
# Total reclaimed space: 2.1GB
```

## Tag: etiquetar imagenes

```bash
# Crear tag para un registro privado
docker tag nginx:latest registry.empresa.com/nginx:1.0

# Crear tag con version
docker tag mi-app:latest mi-app:1.2.3

# Crear multiples tags
docker tag mi-app:latest mi-app:1.2
docker tag mi-app:latest mi-app:1
docker tag mi-app:latest mi-app:latest
```

## Inspect: ver metadatos

```bash
# Ver metadatos completos de una imagen
docker inspect nginx:alpine

# Ver capas
docker history nginx:alpine
# IMAGE          CREATED      CREATED BY                      SIZE
# a1b2c3d4e5f6   2 weeks ago  /bin/sh -c #(nop) CMD ["nginx…  0B
# f6e5d4c3b2a1   2 weeks ago  /bin/sh -c #(nop) COPY file:…   2.58kB
# ...
# 567890abcdef   3 months ago /bin/sh -c #(nop) ADD file:…    7.74MB

# Ver variables de entorno
docker inspect --format='{{.Config.Env}}' nginx:alpine
# [PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin NGINX_VERSION=1.25.4]

# Ver puertos expuestos
docker inspect --format='{{.Config.ExposedPorts}}' nginx:alpine
# map[80/tcp:{}]
```

> Dominar las operaciones basicas con imagenes permite trabajar eficientemente con contenedores: desde descargar imagenes oficiales hasta etiquetar y eliminar las que ya no se necesitan. Son las operaciones mas frecuentes en el dia a dia de cualquier desarrollador o administrador de contenedores.

