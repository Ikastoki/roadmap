# Ejecutar un contenedor

El comando `docker run` es la instruccion fundamental para crear y arrancar contenedores. A diferencia de `docker start`, que reactiva un contenedor existente detenido, `docker run` crea una nueva instancia a partir de una imagen y la ejecuta en un solo paso. Su firma basica combina el nombre de la imagen con una serie de opciones que determinan como se comporta el contenedor: su nombre, los puertos que expone, las variables de entorno que recibe, el modo de ejecucion y los recursos que se le asignan. Cada opcion resuelve un aspecto concreto del aislamiento y la integracion del contenedor con el sistema anfitrion.

```bash
# Ejecutar nginx en segundo plano mapeando el puerto 80 del contenedor al 8080 del host
docker run -d -p 8080:80 nginx
```

## Modo daemon: -d

La opcion `-d` (detach) ejecuta el contenedor en segundo plano. Sin ella, el contenedor se ejecuta en primer plano (foreground) y su salida estandar se muestra directamente en la terminal. En produccion los contenedores se ejecutan casi siempre en modo detached, liberando la terminal para otras tareas. Cuando se usa `-d`, Docker devuelve el ID completo del contenedor y el proceso continua ejecutandose independientemente de la sesion del terminal.

```bash
# Sin -d: la terminal queda bloqueada mostrando el log de nginx
docker run nginx

# Con -d: Docker devuelve el ID y el terminal queda libre
docker run -d nginx
```

## Mapeo de puertos: -p

La opcion `-p` (publish) enlaza un puerto del contenedor con un puerto del anfitrion. El formato es `-p <puerto_host>:<puerto_contenedor>`. Sin este mapeo, el contenedor es accesible solo desde otros contenedores en la misma red Docker, pero no desde el exterior. El puerto del host puede ser cualquiera no utilizado, y si se omite, Docker asigna uno aleatorio.

```bash
# Mapeo explicito
docker run -d -p 8080:80 nginx

# Multiples puertos
docker run -d -p 8080:80 -p 8443:443 nginx

# Puerto aleatorio en el host
docker run -d -p 80 nginx
```

## Nombre del contenedor: --name

Docker asigna un nombre aleatorio a cada contenedor (como `cool_benz` o `mysterious_hopper`), pero es recomendable usar `--name` para asignar un nombre descriptivo. El nombre debe ser unico entre todos los contenedores del sistema, activos o detenidos, y permite referenciar el contenedor en lugar de usar su ID.

```bash
docker run -d --name mi-nginx -p 8080:80 nginx
docker stop mi-nginx
docker rm mi-nginx
```

## Variables de entorno: -e

La opcion `-e` inyecta variables de entorno en el contenedor. Muchas imagenes oficiales utilizan variables de entorno para configurar su comportamiento: contraseñas de bases de datos, nombres de usuario, modos de ejecucion, etc. Las variables se definen con el formato `-e NOMBRE=valor`.

```bash
# Configurar PostgreSQL con variables de entorno
docker run -d --name mi-postgres -e POSTGRES_PASSWORD=secreto -e POSTGRES_DB=miapp postgres:15

# Variables desde el entorno del host (sin valor explicito)
docker run -d --name mi-app -e HOSTNAME -e MI_VAR nginx
```

## Modo interactivo: -it

La combinacion `-it` (`-i` interactivo + `-t` pseudo-TTY) permite ejecutar un contenedor en modo interactivo, util para pruebas, depuracion o shells interactivos. Sin `-t` la terminal no tiene formato adecuado; sin `-i` la entrada estandar no se redirige al contenedor.

```bash
# Shell interactivo dentro de un contenedor que se elimina al salir
docker run -it --rm alpine /bin/sh

# Ejecutar un comando y obtener su salida
docker run alpine echo "Hola mundo"
```

## Exponer puertos: --expose

A diferencia de `-p`, la opcion `--expose` solo documenta que el contenedor escucha en un puerto determinado, pero no lo publica en el host. Es util como metadato para otros desarrolladores o herramientas de orquestacion.

```bash
docker run -d --name api --expose 3000 mi-api
```

## Montar directorios: -v

La opcion `-v` monta un directorio del host o un volumen Docker en el sistema de archivos del contenedor. Es el mecanismo basico para compartir datos entre el host y el contenedor, y para persistir informacion mas alla del ciclo de vida del contenedor.

```bash
# Montar un directorio del host en el contenedor
docker run -d --name web -v /home/user/web:/usr/share/nginx/html:ro -p 8080:80 nginx
```

> El orden de las opciones importa: `docker run [opciones] imagen [comando]`. Todo lo que va despues del nombre de la imagen es el comando que se ejecuta dentro del contenedor, sobrescribiendo el CMD definido en el Dockerfile.

