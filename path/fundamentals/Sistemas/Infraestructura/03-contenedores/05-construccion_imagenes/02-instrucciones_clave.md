# Instrucciones clave del Dockerfile

Cada instruccion del Dockerfile tiene un proposito especifico y genera una capa en la imagen. Comprender cada instruccion permite construir imagenes eficientes, seguras y mantenibles.

## FROM: base de la imagen

```dockerfile
FROM node:20-alpine
# FROM define la imagen base sobre la que se construye
# Siempre debe ser la primera instruccion (excepto ARG antes de FROM)
# Puede haber multiples FROM para multi-stage builds
```

## RUN: ejecutar comandos

```dockerfile
# Instalar paquetes del sistema
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      git \
      && \
    rm -rf /var/lib/apt/lists/*

# Instalar dependencias de Node
RUN npm ci --production

# Crear usuario no root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Crear directorio con permisos
RUN mkdir -p /app && chown appuser:appgroup /app
```

Cada `RUN` crea una capa. Combinar comandos reduce el numero de capas y el tamano de la imagen.

## COPY y ADD: copiar archivos

```dockerfile
# Copiar archivos del contexto al contenedor
COPY package.json ./
COPY src/ ./src/
COPY --chown=appuser:appgroup . .
COPY --chmod=755 scripts/ ./scripts/
```

`ADD` tiene funcionalidades extra:
```dockerfile
# ADD puede extraer tar automaticamente
ADD files.tar.gz /app/

# ADD puede descargar de URLs
ADD https://example.com/file.tar.gz /tmp/
```

En la practica, `COPY` es preferible porque es mas predecible y transparente.

## WORKDIR: directorio de trabajo

```dockerfile
WORKDIR /app
# Crea el directorio si no existe
# Todas las instrucciones siguientes se ejecutan desde ahi
# docker run usa este directorio como base
```

## ENV y ARG: variables

```dockerfile
# ENV: variable de entorno disponible en runtime
ENV NODE_ENV=production
ENV PORT=3000

# ARG: variable solo disponible durante la construccion
ARG BUILD_VERSION=1.0
RUN echo "Building version $BUILD_VERSION"
# No esta disponible en runtime
```

## EXPOSE: documentar puertos

```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 8080/tcp
# Solo documenta puertos, no los publica
# La publicacion se hace con -p en docker run
```

## USER: usuario de ejecucion

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
# El contenedor se ejecuta como appuser
# Si no se define USER, se ejecuta como root (inseguro)
```

## HEALTHCHECK: verificacion de salud

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
# Docker verifica la salud cada 30 segundos
# Si falla 3 veces consecutivas, marca el contenedor como unhealthy
```

## LABEL: metadatos

```dockerfile
LABEL maintainer="equipo@empresa.com"
LABEL version="1.2.3"
LABEL description="API REST para gestion de pedidos"
LABEL org.opencontainers.image.source="https://github.com/empresa/api"
```

## SHELL: cambiar shell por defecto

```dockerfile
# Por defecto en Linux: /bin/sh -c
# Cambiar a bash:
SHELL ["/bin/bash", "-c"]
RUN echo "Hola desde bash" && ls -la
```

> Las instrucciones del Dockerfile son el DNA de la imagen. Cada instruccion bien escrita, optimizada y documentada contribuye a una imagen que es rapida de construir, pequena en tamano y segura en ejecucion.

