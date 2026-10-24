# Conceptos clave de Docker

Para trabajar eficazmente con Docker es necesario dominar un conjunto de conceptos fundamentales que forman el vocabulario basico de la contenerizacion. Cada uno de estos conceptos representa una abstraccion que resuelve un problema especifico: desde como se empaqueta el software hasta como se gestionan los datos persistentes o la comunicacion entre contenedores. Sin una comprension solida de estos conceptos, el uso de Docker se vuelve superficial y propenso a errores.

## Imagen

Una imagen es una plantilla inmodificable y ejecutable que contiene todo lo necesario para ejecutar una aplicacion: un sistema de archivos con el codigo, las bibliotecas, las herramientas del sistema y las configuraciones. Las imagenes se construyen a partir de un Dockerfile y se almacenan en un registro como Docker Hub.

```bash
# Las imagenes son inmutables y se identifican por su digest
docker pull alpine:3.18
docker image inspect alpine:3.18
# Cada imagen tiene un hash SHA256 unico que garantiza su integridad
```

Las imagenes estan compuestas por capas (layers). Cada instruccion en un Dockerfile crea una nueva capa que contiene los cambios respecto a la capa anterior. Las capas son de solo lectura y se comparten entre imagenes, lo que ahorra espacio en disco y acelera las descargas.

```dockerfile
# Cada linea crea una capa
FROM alpine:3.18        # Capa 1: sistema base
RUN apk add --no-cache python3  # Capa 2: Python instalado
COPY app.py /app/       # Capa 3: codigo fuente
CMD ["python3", "/app/app.py"]   # Capa 4: metadata
```

## Contenedor

Un contenedor es una instancia ejecutable de una imagen. Cuando se inicia un contenedor, Docker anade una capa de escritura sobre las capas de solo lectura de la imagen, crea los namespaces y cgroups necesarios, y ejecuta el proceso definido en el CMD o ENTRYPOINT de la imagen.

```bash
# De la imagen al contenedor
docker build -t mi-app .    # Construir la imagen
docker run -d --name app1 mi-app  # Crear y ejecutar un contenedor
docker run -d --name app2 mi-app  # Otro contenedor a partir de la misma imagen
```

Cada contenedor es aislado: tiene su propio sistema de archivos, su propia interfaz de red, su propio arbol de procesos y sus propios mounts. Dos contenedores creados a partir de la misma imagen son completamente independientes entre si.

## Registro

Un registro es un repositorio centralizado donde se almacenan y distribuyen imagenes de contenedores. Docker Hub es el registro publico por defecto, pero existen alternativas como Amazon ECR, Google Container Registry, Azure Container Registry, GitHub Container Registry y registros privados como Harbor o Nexus.

```bash
# Subir y bajar imagenes del registro
docker pull nginx:latest           # Descargar de Docker Hub
docker tag mi-app:1.0 usuario/mi-app:1.0  # Etiquetar para subir
docker push usuario/mi-app:1.0     # Subir al registro
```

Los registros pueden ser publicos o privados. Los registros privados requieren autenticacion y son esenciales para organizaciones que necesitan controlar la distribucion de sus imagenes.

## Dockerfile

Un Dockerfile es un archivo de texto que contiene las instrucciones para construir una imagen. Es la receta que define el entorno de ejecucion de la aplicacion: desde el sistema operativo base hasta el comando que se ejecuta al iniciar el contenedor.

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

El Dockerfile es el punto de partida de cualquier proyecto que use Docker. Su diseno determina el tamano de la imagen, su seguridad y la velocidad de las reconstrucciones.

## Volumen

Un volumen es un mecanismo para persistir datos generados o utilizados por los contenedores. Como los contenedores son efimeros (su capa de escritura se descarta al eliminarlos), los volumenes proporcionan almacenamiento persistente que sobrevive al ciclo de vida del contenedor.

```bash
# Crear y usar un volumen
docker volume create datos-postgres
docker run -d -v datos-postgres:/var/lib/postgresql/data postgres:15

# Los datos persisten aunque el contenedor se elimine
docker rm postgres
docker run -d -v datos-postgres:/var/lib/postgresql/data postgres:15
# Los datos siguen ahi
```

Los volumenes se almacenan en el sistema de archivos del anfitrion, gestionados por Docker, y son la forma recomendada de persistir datos en entornos de produccion.

## Red

Docker proporciona capacidades de red que permiten a los contenedores comunicarse entre si y con el exterior. Cada contenedor tiene su propia interfaz de red virtual, y Docker puede crear redes virtuales aisladas donde los contenedores pueden descubrirse y comunicarse.

```bash
# Crear una red y conectar contenedores
docker network create mi-red
docker run -d --name api --network mi-red mi-api:1.0
docker run -d --name web --network mi-red -p 80:80 nginx
# web puede comunicarse con api usando el nombre del contenedor como hostname
```

Docker ofrece varios tipos de redes: bridge (por defecto, para contenedores en el mismo host), host (comparte la red del anfitrion), overlay (para contenedores en multiples hosts en un swarm), y macvlan (asigna direcciones MAC a los contenedores).

## Docker Compose

Docker Compose es una herramienta para definir y ejecutar aplicaciones multicontenedor. En lugar de ejecutar multiples comandos `docker run`, se define toda la aplicacion en un archivo YAML y se gestiona con un solo comando.

```yaml
services:
  web:
    build: .
    ports:
      - "80:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/app
  db:
    image: postgres:15
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: pass

volumes:
  postgres-data:
```

Docker Compose simplifica la gestion de aplicaciones que requieren multiples servicios, definiendo las dependencias, redes y volumenes de forma declarativa.

## Docker Hub

Docker Hub es el registro publico de imagenes de Docker. Alberga miles de imagenes oficiales (como Ubuntu, Alpine, Nginx, Python, Node.js, PostgreSQL) y millones de imagenes de la comunidad. Es el punto de partida para encontrar imagenes base sobre las que construir las propias aplicaciones.

```bash
# Imagenes oficiales en Docker Hub
docker search python
docker pull python:3.11-slim
docker pull postgres:15
docker pull nginx:1.25-alpine
```

Las imagenes oficiales siguen buenas practicas de seguridad, tienen documentacion detallada y reciben actualizaciones regulares. Usar imagenes oficiales como base reduce la superficie de ataque y simplifica el mantenimiento.

## BuildKit

BuildKit es el motor de construccion moderno de Docker, introducido para reemplazar el builder legacy. Ofrece construcciones paralelas, cache avanzada, secrects seguros y mejor rendimiento general.

```bash
# Usar BuildKit (habilitado por defecto en versiones recientes)
DOCKER_BUILDKIT=1 docker build -t mi-app .

# Construccion con salida progresiva
docker build --progress=plain -t mi-app .
```

BuildKit processa las instrucciones del Dockerfile de forma concurrente cuando es posible, detecta automaticamente las dependencias entre capas y solo reconstruye lo necesario, acelerando significativamente las construcciones.

> Estos conceptos forman la base sobre la que se construye todo el ecosistema Docker. Cada uno resuelve un problema especifico: las imagenes empaquetan el software, los contenedores lo ejecutan, los volumenes persisten los datos, las redes conectan los servicios y los registros distribuyen los artefactos. La comprension profunda de estos conceptos es el requisito previo para disenar arquitecturas robustas con contenedores.

