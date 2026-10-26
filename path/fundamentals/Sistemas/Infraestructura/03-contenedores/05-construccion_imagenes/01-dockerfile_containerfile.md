# Containerfile / Dockerfile

Un Dockerfile (o Containerfile para Podman) es un archivo de texto con instrucciones secuenciales que definen como construir una imagen de contenedor. Cada instruccion genera una capa en la imagen, y el conjunto de capas forma la imagen final. Los Dockerfiles son el equivalente a un script de instalacion pero para contenedores: automatizan la creacion de imagenes reproducibles y versionadas.

## Estructura basica

```dockerfile
# Comentario: este archivo define una imagen Node.js

# 1. Imagen base
FROM node:20-alpine

# 2. Metadatos
LABEL maintainer="equipo@empresa.com"
LABEL version="1.0"

# 3. Directorio de trabajo
WORKDIR /app

# 4. Copiar archivos de dependencias
COPY package.json package-lock.json ./

# 5. Instalar dependencias
RUN npm ci --production

# 6. Copiar codigo fuente
COPY src/ ./src/

# 7. Exponer puerto
EXPOSE 3000

# 8. Comando por defecto
CMD ["node", "src/server.js"]
```

## Instrucciones clave

**FROM: imagen base**

```dockerfile
FROM node:20-alpine
FROM python:3.12-slim
FROM gcr.io/distroless/static-debian12
FROM scratch    # Imagen vacia total (para binarios compilados)
```

**RUN: ejecutar comandos durante la construccion**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    make \
    && rm -rf /var/lib/apt/lists/*

RUN npm ci --production
RUN pip install --no-cache-dir -r requirements.txt
```

**COPY / ADD: copiar archivos**

```dockerfile
COPY src/ ./src/
COPY package.json ./
COPY --chown=appuser:appuser config/ ./config/
```

`ADD` es similar a `COPY` pero puede extraer archivos tar y descargar URLs. En la mayoria de los casos, `COPY` es preferible por ser mas predecible.

**WORKDIR: directorio de trabajo**

```dockerfile
WORKDIR /app
# Todas las instrucciones siguientes se ejecutan desde /app
# Equivale a "mkdir -p /app && cd /app"
```

**ENV: variables de entorno**

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

**EXPOSE: documentar puertos (no publica)**

```dockerfile
EXPOSE 80
EXPOSE 443
# Solo documenta que la app usa estos puertos
# No publica puertos (eso se hace con -p en docker run)
```

**CMD vs ENTRYPOINT:**

```dockerfile
# CMD: comando por defecto, se puede sobreescribir
CMD ["node", "server.js"]
# docker run mi-app           → node server.js
# docker run mi-app bash      → bash (sobreescribe CMD)

# ENTRYPOINT: siempre se ejecuta, los argumentos se añaden
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run mi-app           → node server.js
# docker run mi-app app.js    → node app.js (app.js se añade a ENTRYPOINT)
```

**USER: usuario no root**

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
# El contenedor se ejecuta como appuser, no como root
```

**ARG: argumentos de construccion**

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
# Se puede sobreescribir: docker build --build-arg NODE_VERSION=18 .
```

> Un Dockerfile bien estructurado es la documentacion viva de como se construye una imagen. Las instrucciones ordenadas por frecuencia de cambio, la limpieza de caches y el uso de multi-stage builds son las practicas que convierten un Dockerfile funcional en uno eficiente y mantenible.

