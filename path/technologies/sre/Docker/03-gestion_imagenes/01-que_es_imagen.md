# Que es una imagen de Docker

Una imagen de Docker es una plantilla inmutable y portable que contiene todo lo necesario para ejecutar una aplicacion: codigo, runtime, bibliotecas, variables de entorno y archivos de configuracion. A diferencia de una maquina virtual que incluye un sistema operativo completo con su propio kernel, una imagen Docker comparte el kernel del anfitrion y solo incorpora las capas de usuario necesarias. Esta distincion es la que permite que las imagenes sean ordenes de magnitud mas pequenas que los discos virtuales tradicionales y que su descarga,almacenamiento y ejecucion sean practicamente instantaneos.

## Naturaleza inmutable

Una vez construida, una imagen no puede modificarse. No existe un comando para "editar" una imagen; cualquier cambio requiere construir una nueva imagen a partir de un Dockerfile o de un contenedor en ejecucion mediante `docker commit`. Esta inmutabilidad es una propiedad de diseno fundamental que garantiza que la imagen que se prueba en desarrollo sea exactamente la misma que se despliega en produccion, eliminando la desviacion de configuracion entre entornos.

```bash
# Intentar modificar una imagen directamente no es posible
# La unica forma de generar una nueva imagen es:
docker commit contenedor_modificado nueva-imagen:v1
```

La inmutabilidad tambien tiene implicaciones de seguridad: si se descubre una vulnerabilidad en una imagen, no basta con parchear el contenedor en ejecucion, sino que hay que construir una nueva imagen, distribuirla y reemplazar los contenedores activos. Este flujo, aunque mas riguroso, fuerza un proceso de actualizacion controlado y trazable.

## Sistema de capas apiladas

Las imagenes Docker no son bloques monoliticos, sino que se construyen como una pila de capas de solo lectura. Cada instruccion en un Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`) genera una nueva capa que contiene los cambios introducidos respecto al estado anterior. Docker utiliza un sistema de archivos por capas (tipicamente overlay2) para superponer estas capas y presentarlas como un unico arbol de archivos coherente.

```bash
# Ver las capas que componen una imagen
docker history nginx:latest

# La salida muestra cada capa con su ID, tamano y comando que la creo
IMAGE          CREATED       CREATED BY                                      SIZE
a6b72b9c8e1e   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]               0B
<missing>      2 weeks ago   STOPSIGNAL SIGQUIT                             0B
<missing>      2 weeks ago   EXPOSE 80                                      0B
<missing>      2 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]           0B
<missing>      2 weeks ago   COPY docker-entrypoint.sh /                    1.2kB
<missing>      2 weeks ago   RUN /bin/sh -c set -eux; ...                   112MB
<missing>      2 weeks ago   ENV NGINX_VERSION=1.25.3                       0B
<missing>      2 weeks ago   apt-get update && apt-get install ...           87MB
```

Cada capa se identifica mediante un hash SHA-256 y se almacena en el directorio `/var/lib/docker/overlay2/`. Las capas se cachean y reutilizan entre imagenes: si dos imagenes diferentes comparten las mismas capas base, estas se almacenan una sola vez en disco, independientemente de cuantas imagenes las referencien.

## Capa de escritura del contenedor

Cuando se crea un contenedor a partir de una imagen, Docker anade una capa de escritura adicional sobre la pila de capas de solo lectura. Esta capa, conocida como _container layer_, es donde se registran todos los cambios que ocurren durante la ejecucion del contenedor: creacion de archivos, modificaciones, eliminaciones. Cuando el contenedor se elimina, esta capa se descarta a menos que se haya configurado un volumen para persistir los datos.

```bash
# El tamano de la capa de escritura se puede consultar
docker ps --size

# Ver el detalle del sistema de archivos de un contenedor
docker inspect --format '{{.GraphDriver}}' contenedor
```

> El sistema de copia en escritura (copy-on-write) de Docker garantiza que las capas base nunca se modifiquen. Cuando un contenedor intenta escribir en un archivo que pertenece a una capa de solo lectura, el sistema copia ese archivo a la capa de escritura y aplica la modificacion alli. Esto es transparente para el proceso del contenedor.

## Portabilidad y estandarizacion

Una imagen Docker sigue las especificaciones de la Open Container Initiative (OCI), lo que garantiza que pueda ejecutarse en cualquier runtime compatible con OCI, no solo en Docker. Esta estandarizacion ha permitido que el mismo formato de imagen sea utilizado por contenedores en entornos de desarrollo, servidores on-premise, nubes publicas y dispositivos edge.

```bash
# Una imagen se puede exportar como archivo tar para transporte offline
docker save nginx:latest > nginx.tar

# Cargar la imagen en otro sistema
docker load < nginx.tar
```

La portabilidad no es solo tecnica, sino tambien operativa: una imagen construida en un equipo Linux con arquitectura x86_64 funcionara en cualquier otro sistema Linux x86_64 con Docker, sin necesidad de instalar dependencias ni resolver conflictos de versiones. Para otras arquitecturas, Docker soporta multiplataforma mediante emulacion con QEMU o compilacion cruzada.

```bash
# Construir una imagen para multiples arquitecturas
docker buildx build --platform linux/amd64,linux/arm64 --push -t usuario/app:latest .
```

