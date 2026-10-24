# Instrucciones clave del Dockerfile

Un Dockerfile se compone de instrucciones, cada una con una funcion especifica en la construccion de la imagen. La mayoria de estas instrucciones generan una capa, pero algunas solo anaden metadatos a la configuracion de la imagen. Conocer el comportamiento exacto de cada una es esencial para escribir Dockerfiles eficientes y predecibles.

## FROM: imagen base

`FROM` es la primera instruccion de un Dockerfile funcional y establece la imagen base sobre la que se construiran todas las capas posteriores. Puede referirse a una imagen oficial, una imagen de un registro privado o incluso a una etapa anterior en un multi-stage build.

```dockerfile
FROM python:3.11-slim
FROM alpine:3.18
FROM registry.internal.com/mi-base:1.0
```

Una imagen base debe incluir al menos el sistema de archivos minimo y el gestor de paquetes necesario para instalar dependencias. La eleccion de la imagen base es la decision mas importante en terminos de seguridad y tamano de la imagen final.

> Alpine Linux es una base popular por su tamano reducido (~5 MB), pero usa musl en lugar de glibc, lo que puede causar incompatibilidades con algunas bibliotecas compiladas para glibc.

## RUN: ejecutar comandos durante el build

`RUN` ejecuta comandos en una capa intermedia y commitea el resultado. Es la instruccion principal para instalar paquetes, crear directorios, compilar binarios y cualquier otra operacion que modifique el sistema de archivos de la imagen.

```dockerfile
RUN apt-get update && apt-get install -y curl
RUN pip install --no-cache-dir -r requirements.txt
RUN mkdir -p /app/data && chown -R appuser:appuser /app
```

Cada `RUN` crea una nueva capa. Si un comando genera archivos temporales que se eliminan en un `RUN` posterior, esos archivos siguen ocupando espacio en la capa intermedia. Por eso se recomienda encadenar operaciones relacionadas en un solo `RUN`.

```dockerfile
# Optimizado: los archivos temporales nunca persisten en una capa
RUN apt-get update && apt-get install -y build-essential \
    && pip install --no-cache-dir -r requirements.txt \
    && apt-get remove -y build-essential && apt-get autoremove -y
```

## COPY y ADD: transferir archivos al contenedor

`COPY` copia archivos y directorios desde el contexto de build al sistema de archivos de la imagen. Es la instruccion preferida para transferir codigo fuente, scripts y archivos estaticos porque su comportamiento es predecible y explicito.

```dockerfile
COPY requirements.txt /app/
COPY . /app/
COPY --chown=appuser:appuser config/ /app/config/
```

`ADD` es una version extendida que soporta dos caracteristicas adicionales: descompresion automatica de archivos tar y URLs remotas. Sin embargo, estas capacidades introducen comportamientos sorprendentes y son fuente de problemas de seguridad, por lo que la practica recomendada es usar `COPY` a menos que se necesite explcitamente la descompresion automatica.

```dockerfile
# ADD descomprime automaticamente el tar
ADD app.tar.gz /app/
```

> Usar `ADD` con URLs remotas es una mala practica porque la descarga ocurre durante el build sin posibilidad de cachear la descarga de forma eficiente. Es preferible usar `curl` o `wget` dentro de un `RUN`.

## WORKDIR: directorio de trabajo

`WORKDIR` establece el directorio de trabajo para todas las instrucciones posteriores: `RUN`, `CMD`, `ENTRYPOINT`, `COPY` y `ADD`. Si el directorio no existe, se crea automaticamente.

```dockerfile
WORKDIR /app
COPY . .          # Copia al directorio /app
RUN pwd           # Muestra /app
```

Es equivalente a ejecutar `mkdir -p /app && cd /app`, pero expresado de forma declarativa y sin generar una capa adicional. Docker recomienda usar `WORKDIR` en lugar de `cd` en los comandos `RUN` porque el efecto de `cd` se pierde entre instrucciones.

## ENV: variables de entorno

`ENV` define variables de entorno que estaran disponibles durante el build y en los contenedores que se ejecuten a partir de la imagen. Se almacenan en la configuracion de la imagen y son heredadas por todos los procesos hijos.

```dockerfile
ENV PYTHONUNBUFFERED=1
ENV APP_HOME=/app
ENV PORT=8000
```

Las variables definidas con `ENV` persisten en la imagen final. Si una variable contiene informacion sensible (claves API, tokens), esta quedara visible en los metadatos de la imagen y en el contenedor en ejecucion. Para valores sensibles que solo se necesitan durante el build, se usa `ARG`.

```dockerfile
ARG DEBIAN_FRONTEND=noninteractive   # Solo durante el build
ENV DATABASE_URL=postgres://...       # Persiste en la imagen (no recomendado para secretos)
```

## EXPOSE: documentar puertos

`EXPOSE` indica que la imagen escuchara en un puerto especifico en tiempo de ejecucion. Es una instruccion puramente documental: no publica el puerto ni lo hace accesible desde el exterior. Docker la usa como metadato que puede consultarse con `docker inspect`.

```dockerfile
EXPOSE 8000
EXPOSE 443/tcp
```

Para que un puerto sea accesible desde el host, es necesario usar `-p` o `-P` en `docker run`. `-P` publica todos los puertos declarados con `EXPOSE` en puertos efimeros del host.

```bash
docker run -P mi-app:1.0          # Publica EXPOSE 8000 en un puerto aleatorio
docker run -p 8000:8000 mi-app:1.0  # Publicacion explicita
```

## CMD y ENTRYPOINT: el comando por defecto

`CMD` especifica el comando por defecto que se ejecutara al iniciar el contenedor. Si no se proporciona ningun comando en `docker run`, se ejecuta `CMD`. Si se proporciona un comando, este reemplaza a `CMD` por completo.

`ENTRYPOINT` configura el ejecutable principal del contenedor. A diferencia de `CMD`, los argumentos pasados en `docker run` no reemplazan a `ENTRYPOINT`, sino que se concatenan como argumentos.

```dockerfile
# Formas de CMD
CMD ["python", "app.py"]           # Forma exec (recomendada)
CMD python app.py                  # Forma shell
CMD ["python", "app.py", "--port", "8000"]

# ENTRYPOINT fijo con CMD como argumentos por defecto
ENTRYPOINT ["python"]
CMD ["app.py"]
```

La combinacion mas util es `ENTRYPOINT` como ejecutable fijo y `CMD` como argumentos por defecto. Esto permite que el usuario sobrescriba los argumentos sin cambiar el ejecutable.

```dockerfile
ENTRYPOINT ["gunicorn"]
CMD ["-w", "4", "-b", "0.0.0.0:8000", "app:app"]
```

```bash
# Usa los argumentos por defecto
docker run mi-app:1.0
# Sobrescribe los argumentos
docker run mi-app:1.0 -w 8 -b 0.0.0.0:9000 app:app
```

> La forma exec (array JSON) es la unica que garantiza que el proceso reciba correctamente las senales del sistema (SIGTERM, SIGINT). La forma shell ejecuta el comando como `/bin/sh -c "comando"`, lo que impide que las senales lleguen al proceso real.

## Orden de las instrucciones

El orden de las instrucciones determina la eficiencia de la cache de capas. Las capas que cambian con menor frecuencia deben colocarse al principio, y las que cambian con mayor frecuencia al final.

```dockerfile
# Orden optimo
FROM python:3.11-slim        # Cambia solo si se actualiza la version
WORKDIR /app                 # Rara vez cambia
COPY requirements.txt .      # Cambia solo con nuevas dependencias
RUN pip install -r requirements.txt  # Se reconstruye solo si cambia requirements.txt
COPY . .                    # Cambia en cada commit
CMD ["python", "app.py"]     # Metadato, no genera capa de contenido
```

