# Gestion de volumenes

Docker proporciona un conjunto de comandos dedicados a la gestion del ciclo de vida de los volumenes, independientes de los contenedores que los utilizan. Estos comandos permiten crear, listar, inspeccionar y eliminar volumenes sin necesidad de manipular el sistema de archivos del host directamente. Ademas, la integracion con `docker run` y `docker-compose.yml` permite asociar volumenes a contenedores de forma declarativa.

## Creacion de volumenes

El comando `docker volume create` crea un volumen con un nombre explicito. Si no se proporciona nombre, Docker genera un identificador hexadecimal unico. Es posible especificar opciones adicionales como el driver (por defecto `local`), etiquetas de metadatos y parametros especificos del driver.

```bash
# Crear un volumen nombrado
docker volume create datos-postgres

# Crear un volumen con etiquetas
docker volume create --label proyecto=miapp --label entorno=produccion datos-prod

# Crear un volumen con driver NFS (requiere configuracion previa)
docker volume create --driver nfs --opt type=nfs --opt o=addr=192.168.1.100,rw --opt device=:/export/data nfs-volumen
```

Cuando no se crea el volumen explicitamente antes de usarlo, Docker lo crea de forma implicita en el momento de ejecutar el contenedor si el nombre del volumen no existe. Este comportamiento automatico es comodo pero puede llevar a confusion si se escribe mal el nombre, ya que Docker creara un volumen nuevo sin advertirlo.

## Montaje de volumenes en contenedores

El montaje se realiza mediante las banderas `-v` o `--mount` en `docker run`. La sintaxis corta `-v` sigue el formato `nombre_volumen:ruta_destino:opciones`. La sintaxis larga `--mount` es mas explicita y se recomienda en scripts de produccion.

```bash
# Sintaxis corta: volumen nombrado
docker run -d --name postgres \
  -v datos-postgres:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secreto \
  postgres:16

# Sintaxis larga (recomendada para claridad)
docker run -d --name postgres \
  --mount type=volume,source=datos-postgres,target=/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secreto \
  postgres:16

# Montaje de solo lectura
docker run -d --name nginx \
  -v datos-estaticos:/usr/share/nginx/html:ro \
  nginx:alpine
```

La opcion `:ro` al final del montaje establece el volumen como solo lectura. Esto es util cuando un contenedor necesita acceder a datos que otro servicio gestiona, evitando modificaciones accidentales. La opcion `:z` y `:Z` son especificas de SELinux y controlan el etiquetado de seguridad del volumen montado.

## Listado e inspeccion

`docker volume ls` enumera todos los volumenes presentes en el host. Admite filtros por nombre, driver, etiqueta y estado. `docker volume inspect` muestra los metadatos detallados de uno o varios volumenes, incluyendo su punto de montaje en el sistema de archivos del host, el driver utilizado y las opciones de configuracion.

```bash
# Listar todos los volumenes
docker volume ls

# Filtrar por etiqueta
docker volume ls --filter label=proyecto=miapp

# Filtrar volumenes huerfanos (no usados por ningun contenedor)
docker volume ls --filter dangling=true

# Inspeccionar un volumen
docker volume inspect datos-postgres
```

La salida de `docker volume inspect` incluye el campo `Mountpoint`, que indica la ruta absoluta en el host donde Docker almacena los datos del volumen (tipicamente `/var/lib/docker/volumes/datos-postgres/_data`). Aunque es posible acceder a esta ruta directamente, hacerlo sin la mediacion de Docker puede causar inconsistencias si el volumen esta en uso.

## Eliminacion de volumenes

`docker volume rm` elimina uno o varios volumenes. Solo se pueden eliminar volumenes que no esten siendo utilizados por ningun contenedor, ya sea en ejecucion o detenido. Si se intenta eliminar un volumen en uso, Docker devuelve un error.

```bash
# Eliminar un volumen
docker volume rm datos-postgres

# Eliminar varios volumenes
docker volume rm vol1 vol2 vol3

# Eliminar todos los volumenes no utilizados
docker volume prune

# Eliminar volumenes no utilizados con confirmacion automatica
docker volume prune -f
```

`docker volume prune` es especialmente util para liberar espacio en disco, ya que los volumenes huerfanos pueden acumularse con el tiempo. Por defecto, Docker no elimina los volumenes automaticamente al eliminar el contenedor, a diferencia de lo que ocurre con las redes o los contenedores detenidos.

## Volumenes en Docker Compose

En un archivo `docker-compose.yml`, los volumenes nombrados se declaran en la seccion `volumes:` del nivel superior y se referencian desde los servicios mediante la clave `volumes:` de cada servicio. Los volumenes anonimos se pueden definir directamente en el servicio sin declararlos en el nivel superior.

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16
    volumes:
      - datos-postgres:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    environment:
      POSTGRES_PASSWORD: secreto

  app:
    image: miapp:latest
    volumes:
      - datos-compartidos:/uploads
    depends_on:
      - postgres

volumes:
  datos-postgres:
    driver: local
  datos-compartidos:
```

Cuando se ejecuta `docker compose up -d` por primera vez, Docker crea los volumenes declarados en la seccion `volumes:` con el nombre compuesto por el prefijo del proyecto y el nombre del volumen (por ejemplo, `miapp_datos-postgres`). Al usar `docker compose down`, los contenedores se eliminan pero los volumenes permanecen a menos que se anada la bandera `-v`.

> No usar `docker compose down -v` en produccion a menos que se tenga la intencion explicita de destruir los datos persistentes. Esta bandera elimina los volumenes junto con los contenedores y las redes.

