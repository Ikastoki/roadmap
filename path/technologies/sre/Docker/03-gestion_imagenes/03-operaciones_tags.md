# Operaciones basicas y tags

La gestion de imagenes en Docker se articula en torno a un conjunto de operaciones fundamentales que permiten descargar, listar, eliminar y versionar imagenes. Dominar estos comandos y comprender el sistema de etiquetado es esencial para trabajar con contenedores de forma eficiente y evitar problemas tipicos de entornos productivos.

## Descarga de imagenes: docker pull

El comando `docker pull` descarga una imagen desde un registro a la maquina local. Si no se especifica un registro, Docker busca en Docker Hub por defecto. La descarga solo transfiere las capas que no existen localmente, gracias al sistema de cache de capas.

```bash
# Descargar la ultima version de nginx
docker pull nginx

# Descargar una version especifica
docker pull nginx:1.25.3

# Descargar desde un registro distinto a Docker Hub
docker pull registry.example.com/mi-app:v2

# Descargar con un tag que incluye la variante alpine
docker pull nginx:1.25-alpine
```

Internamente, `docker pull` consulta el manifiesto de la imagen en el registro, que contiene la lista de capas con sus hashes SHA-256. Docker compara esos hashes con las capas almacenadas localmente en `/var/lib/docker/overlay2/` y solo descarga las que faltan.

## Listado de imagenes locales: docker images

El comando `docker images` (o `docker image ls`) muestra todas las imagenes descargadas en el sistema, junto con su repositorio, tag, ID de imagen y tamano.

```bash
docker images

REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
nginx         latest    a6b72b9c8e1e   2 weeks ago    187MB
nginx         1.25      3f8a00f3df1e   2 weeks ago    187MB
alpine        latest    05455a0884ea   4 weeks ago    7.05MB
ubuntu        22.04     2dc39ba059dc   2 months ago   77.8MB
```

El campo `IMAGE ID` es el hash SHA-256 de la imagen, truncado a 12 caracteres. Este ID identifica de forma unica la imagen a nivel de contenido. Dos imagenes con el mismo ID son exactamente la misma imagen, independientemente del nombre o tag que tengan asignados.

```bash
# Listar solo los IDs de las imagenes (util para scripting)
docker images -q

# Listar imagenes con formato personalizado
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Filtrar imagenes por repositorio
docker images nginx

# Filtrar imagenes por tag
docker images --filter "reference=*:latest"
```

## Eliminacion de imagenes: docker rmi

El comando `docker rmi` (remove image) elimina una o mas imagenes del almacen local. Solo se puede eliminar una imagen si ningun contenedor (activo o detenido) la referencia.

```bash
# Eliminar una imagen por su tag
docker rmi nginx:1.25

# Eliminar una imagen por su ID
docker rmi a6b72b9c8e1e

# Eliminar varias imagenes a la vez
docker rmi nginx:1.25 ubuntu:22.04

# Forzar la eliminacion aunque haya contenedores que la referencien
docker rmi -f nginx:latest
```

Las imagenes que no son referenciadas por ningun tag ni contenedor se denominan "imagenes colgantes" (dangling images). Suelen generarse al reconstruir una imagen con el mismo tag: el tag se reasigna a la nueva imagen, y la anterior queda sin referencia.

```bash
# Listar imagenes colgantes
docker images --filter "dangling=true"

# Eliminar todas las imagenes colgantes
docker image prune

# Eliminar todas las imagenes no utilizadas (mas agresivo)
docker image prune -a
```

## Tags y versionado

Los tags son etiquetas textuales que apuntan a un ID de imagen concreto. Un tag no es mas que un puntero: puede reasignarse, moverse o eliminarse sin alterar la imagen subyacente. La misma imagen puede tener multiples tags simultaneamente.

```bash
# Asignar un tag a una imagen existente
docker tag nginx:latest nginx:estable

# Asignar multiples tags
docker tag nginx:latest nginx:1.25 nginx:1.25.3

# Verificar que todos apuntan al mismo ID
docker images --filter "reference=nginx"
```

El tag `latest` es el que se asigna por defecto cuando no se especifica ninguno. Sin embargo, `latest` no significa "la version mas reciente" de forma automatica: es simplemente un tag que los mantenedores actualizan manualmente al publicar una nueva version. Es un error comun asumir que `latest` siempre apunta a la version estable mas moderna.

```bash
# Estas dos operaciones son equivalentes
docker pull nginx
docker pull nginx:latest
```

> En produccion, jamas debe usarse `latest`. El tag `latest` es un puntero movil que cambia con el tiempo, lo que rompe la reproducibilidad: el mismo comando `docker pull nginx:latest` puede descargar imagenes diferentes en momentos distintos. Siempre debe especificarse una version concreta: `nginx:1.25.3`, `postgres:16-alpine`, `python:3.12-slim`.

### Estrategias de versionado

Las imagenes oficiales suelen ofrecer una jerarquia de tags que permite elegir el nivel de precision deseado:

```text
nginx:1.25.3      -> version exacta (inmutable)
nginx:1.25        -> apunta a la ultima revision de la rama 1.25 (movil)
nginx:1           -> apunta a la ultima version mayor 1.x (movil)
nginx:latest      -> apunta a la ultima version estable (movil)
```

A estas variantes se les anade a menudo el sufijo de la distribucion base:

```text
nginx:1.25.3-alpine     -> sobre Alpine Linux (ligera)
nginx:1.25.3-slim       -> sobre Debian minimo
nginx:1.25.3-bookworm   -> sobre Debian Bookworm
nginx:1.25.3-perl       -> con soporte para Perl incluido
```

Elegir el tag adecuado depende del equilibrio entre estabilidad y seguridad. Un tag fijo como `nginx:1.25.3` garantiza que todos los despliegues usen exactamente la misma imagen, pero no recibe parches de seguridad automaticamente. Un tag movil como `nginx:1.25` recibe actualizaciones de revision, pero puede introducir cambios inesperados.

```bash
# Verificar a que ID apunta un tag antes de usarlo
docker pull nginx:1.25.3
docker images nginx:1.25.3

# Si se necesita actualizar, se cambia explicitamente el tag
docker pull nginx:1.25.4
```

## El ciclo de vida completo

El flujo tipico de trabajo con imagenes abarca todas estas operaciones:

```bash
# 1. Buscar y descargar
docker search postgres
docker pull postgres:16-alpine

# 2. Verificar la descarga
docker images postgres

# 3. Etiquetar para uso local
docker tag postgres:16-alpine mi-db:pruebas

# 4. Eliminar cuando ya no sea necesaria
docker rmi postgres:16-alpine mi-db:pruebas

# 5. Limpiar imagenes residuales
docker image prune
```

