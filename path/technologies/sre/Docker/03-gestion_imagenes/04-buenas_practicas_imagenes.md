# Inspeccion y buenas practicas con imagenes

La gestion eficiente de imagenes no termina en conocer los comandos basicos. Construir y mantener imagenes seguras, pequenas y rapidas requiere un conjunto de tecnicas y herramientas que van desde la inspeccion de capas hasta la seleccion cuidadosa de la imagen base. Estas practicas determinan en gran medida la velocidad de los despliegues, la superficie de ataque de los contenedores y el consumo de recursos en los nodos de produccion.

## Inspeccion de imagenes: docker inspect

El comando `docker inspect` devuelve en formato JSON toda la metainformacion de una imagen: capas, variables de entorno, puertos expuestos, entrypoint, volumenes y el historial de construccion. Es la herramienta de diagnostico por excelencia.

```bash
# Inspeccion completa de una imagen
docker inspect nginx:1.25

# Extraer campos especificos con --format y plantillas Go
docker inspect --format '{{.Created}}' nginx:1.25
docker inspect --format '{{.ContainerConfig.Env}}' nginx:1.25
docker inspect --format '{{.RootFS.Layers}}' nginx:1.25
```

Los datos mas relevantes que ofrece `docker inspect` son:

- `RootFS.Layers`: lista de hashes SHA-256 de cada capa que compone la imagen.
- `ContainerConfig.Cmd`: comando por defecto que ejecutara el contenedor.
- `ContainerConfig.ExposedPorts`: puertos que la imagen expone.
- `ContainerConfig.Env`: variables de entorno definidas en la imagen.
- `History`: registro de las instrucciones que generaron la imagen, aunque `docker history` ofrece esta informacion de forma mas legible.

## Analisis visual de capas con dive

`dive` es una herramienta de terceros que permite explorar interactivamente las capas de una imagen. Muestra por cada capa los archivos anadidos, modificados o eliminados, el tamano que ocupan y el porcentaje de eficiencia de la imagen.

```bash
# Instalar dive (Linux)
wget https://github.com/wagoodman/dive/releases/download/v0.12.0/dive_0.12.0_linux_amd64.tar.gz
tar xzf dive_0.12.0_linux_amd64.tar.gz && sudo install dive /usr/local/bin/

# Analizar una imagen
dive nginx:1.25
```

La interfaz de dive muestra dos paneles: el izquierdo con el arbol de capas y el derecho con el sistema de archivos. Al seleccionar una capa, se resaltan los cambios que introdujo. La metrica de eficiencia indica que porcentaje de la imagen es realmente utilizado; un valor bajo sugiere que la imagen contiene archivos innecesarios que podrian eliminarse.

> dive resulta especialmente util para auditar imagenes de terceros: permite verificar que no contienen archivos inesperados, como claves SSH, tokens hardcodeados o binarios sospechosos.

## Seleccion de imagenes base

La eleccion de la imagen base es la decision que mas impacto tiene en el tamano final, la seguridad y el rendimiento de la imagen. Las opciones mas comunes se situan en un espectro que va desde minimas hasta completas:

| Imagen base    | Tamano aprox | Contenido                                  | Caso de uso                            |
|----------------|-------------|--------------------------------------------|----------------------------------------|
| scratch        | 0 B         | Sistema de archivos vacio                  | Binarios estaticos compilados          |
| alpine         | 5 MB        | BusyBox + musl libc + apk                  | Aplicaciones ligeras y estaticas       |
| slim           | 40-80 MB    | Debian minimo                              | Runtime de Python, Node, etc.          |
| ubuntu/debian  | 80-200 MB   | Sistema completo con apt y herramientas    | Entornos que requieren paquetes nativos|

`scratch` es una imagen vacia, ideal para binarios compilados estaticamente (Go, Rust, C con musl). No contiene shell, gestor de paquetes ni ninguna utilidad del sistema, lo que minimiza la superficie de ataque.

```dockerfile
FROM scratch
COPY mi-binario /mi-binario
ENTRYPOINT ["/mi-binario"]
```

`alpine` es la opcion mas popular para imagenes ligeras. Utiliza musl libc en lugar de glibc, lo que puede causar incompatibilidades con algunas aplicaciones que dependen de caracteristicas especificas de glibc. En esos casos, las variantes `slim` ofrecen un compromiso razonable.

```bash
# Comparar tamanos de la misma aplicacion sobre diferentes bases
docker images node

REPOSITORY   TAG         IMAGE ID       SIZE
node         22-alpine   8a5c3c0e1d2f   130MB
node         22-slim     9b6d4d1f2e3f   240MB
node         22-bookworm a7c5b4e3d2f1   1.1GB
```

> Las imagenes `distroless` mantenidas por Google llevan la filosofia de scratch un paso mas alla: proporcionan el runtime necesario (Java, Python, Node) sin shell ni gestor de paquetes. Son ultra-ligeras y seguras, aunque dificultan la depuracion en ejecucion al no incluir herramientas como bash o curl.

## Reduccion de capas innecesarias

Cada instruccion en un Dockerfile genera una capa. Aunque las capas se cachean y reutilizan, un exceso de capas incrementa el tamano de la imagen y el tiempo de construccion. La estrategia consiste en minimizar el numero de capas sin sacrificar la legibilidad ni la cacheabilidad.

La practica mas efectiva es combinar instrucciones `RUN` relacionadas en una sola capa, limpiando los archivos temporales en la misma instruccion para que no queden registrados en la capa:

```dockerfile
# MAL: tres capas, archivos de apt en la imagen final
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# BIEN: una sola capa, cache de apt eliminada en la misma capa
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

La instalacion de paquetes y su limpieza deben ocurrir en la misma capa porque las capas son inmutables. Si se eliminan archivos en una capa posterior, las capas anteriores siguen ocupando espacio en el almacenamiento local, aunque los archivos eliminados no sean visibles en el sistema de archivos final.

```bash
# Verificar el tamano real que ocupan las capas
docker history --format "table {{.CreatedBy}}\t{{.Size}}" mi-app
```

## Orden optimo de instrucciones

Docker cachea cada capa y reutiliza la cache mientras el contexto de la instruccion no cambie. Para maximizar el aprovechamiento de la cache, las instrucciones deben ordenarse de menor a mayor frecuencia de cambio:

1. `FROM`: siempre primero, cambia raramente.
2. `ENV`, `ARG`, `LABEL`: cambios poco frecuentes.
3. `RUN apt-get ...`: dependencias del sistema, cambian ocasionalmente.
4. `COPY requirements.txt` + `RUN pip install`: las dependencias de la aplicacion cambian menos que el codigo.
5. `COPY . .`: el codigo fuente, que cambia constantemente.

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

# Dependencias primero (cambian poco)
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Codigo despues (cambia frecuentemente)
COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

Este orden garantiza que, mientras no cambien las dependencias, Docker reutilice las capas cacheadas y solo reconstruya las ultimas capas cuando se modifique el codigo.

## Multi-stage builds

Las construcciones multi-etapa permiten utilizar multiples imagenes base en un mismo Dockerfile y copiar selectivamente archivos de una etapa a otra. Esto es especialmente util para compilar aplicaciones en una imagen completa con todas las herramientas de build y luego copiar el binario resultante a una imagen minima de produccion.

```dockerfile
# Etapa de compilacion
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o servidor .

# Etapa de produccion
FROM alpine:3.20
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/servidor /servidor
EXPOSE 8080
CMD ["/servidor"]
```

La imagen final solo contiene el binario compilado y las certificados CA, sin el compilador Go, el gestor de paquetes ni el codigo fuente. El resultado es una imagen de pocos megabytes en lugar de cientos.

## Etiquetado para auditoria

Adoptar un esquema de etiquetado consistente facilita la trazabilidad de las imagenes en entornos productivos. Ademas del tag de version, es recomendable incluir el hash del commit de Git y la fecha de construccion:

```bash
# Etiquetado con hash de Git y timestamp
docker build -t mi-app:1.0.0 -t mi-app:$(git rev-parse --short HEAD) .
```

En combinacion con `docker inspect`, estas etiquetas permiten rastrear que version del codigo genero cada imagen y cuando se construyo.

```bash
docker inspect --format '{{.RepoTags}}' mi-app:latest
```

## Analisis de vulnerabilidades

Las imagenes deben escanearse en busca de vulnerabilidades conocidas antes de desplegarse. Herramientas como `trivy`, `grype` o `docker scout` (integrado en Docker Desktop) analizan las capas de la imagen y las comparan con bases de datos de vulnerabilidades (CVE).

```bash
# Escanear con trivy
trivy image nginx:1.25

# Escanear con docker scout
docker scout quickview nginx:1.25
```

Integrar el escaneo en el pipeline CI/CD permite rechazar automaticamente imagenes que superen un umbral de vulnerabilidades criticas, evitando que lleguen a produccion.

> Una imagen pequena no es necesariamente una imagen segura, pero reduce la superficie de ataque. Alpine, por ejemplo, expone menos binarios que Ubuntu, lo que se traduce en menos potenciales vulnerabilidades. Sin embargo, la principal fuente de riesgos son las dependencias de la aplicacion, no las herramientas del sistema.

