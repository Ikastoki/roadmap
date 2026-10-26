# Errores comunes de construccion

Los errores en Dockerfiles son frecuentes y pueden generar imagenes inseguras, ineficientes o que simplemente no funcionan. Conocer los errores mas comunes permite evitarlos y construir imagenes robustas desde el primer dia.

## 1. Secretos en la imagen

```dockerfile
# PELIGROSO: secreto hardcodeado
ENV API_KEY=sk-1234567890abcdef
RUN pip install --index-url https://$API_KEY@pypi.empresa.com/ paquete

# Cualquiera que haga docker inspect vera la clave
# La clave queda en el historial de capas permanentemente

# SOLUCION: usar build secrets
# Docker BuildKit:
# RUN --mount=type=secret,id=pypi_key \
#     pip install --index-url https://$(cat /run/secrets/pypi_key)@pypi.empresa.com/ paquete

# docker build --secret id=pypi_key,src=./secrets/pypi_key.txt -t mi-app .
```

## 2. Ejecutar como root

```dockerfile
# MAL: el contenedor se ejecuta como root
FROM node:20-alpine
COPY . .
RUN npm ci
CMD ["node", "server.js"]
# El proceso principal tiene privilegios de root

# BIEN: definir usuario no root
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
RUN npm ci
USER appuser
CMD ["node", "server.js"]
```

## 3. No usar --no-install-recommends

```dockerfile
# MAL: instala paquetes innecesarios
RUN apt-get update && apt-get install -y curl

# BIEN: solo instalar lo necesario
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
# reduce el tamano en 20-50%
```

## 4. No limpiar cache de gestores de paquetes

```dockerfile
# MAL: cache de apt queda en la imagen
RUN apt-get update && apt-get install -y python3
# +20MB de cache innecesario

# BIEN: limpiar en el mismo paso
RUN apt-get update && \
    apt-get install -y python3 && \
    rm -rf /var/lib/apt/lists/*
```

## 5. Copiar archivos grandes innecesarios

```dockerfile
# MAL: copiar todo el proyecto
COPY . .
# Incluye .git, node_modules, tests, docs...

# BIEN: usar .dockerignore y copiar selectivamente
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
COPY public/ ./public/
```

## 6. No especificar version de imagen base

```dockerfile
# MAL: depende de "latest" que puede cambiar
FROM node

# BIEN: especificar version exacta
FROM node:20.11-alpine
```

## 7. Olvidar HEALTHCHECK

```dockerfile
# Sin healthcheck, el orquestador no sabe si la app esta funcionando
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

## 8. Demasiadas capas

```dockerfile
# MAL: 8 capas para 4 operaciones
RUN apt-get update
RUN apt-get install -y python3
RUN pip install flask
RUN pip install sqlalchemy
RUN mkdir /app
COPY . .
RUN chown -R appuser /app
USER appuser

# BIEN: 3 capas para las mismas operaciones
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 && \
    rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir flask sqlalchemy
COPY --chown=appuser:appgroup . /app
```

## Verificar la imagen despues de construccion

```bash
# Verificar que se ejecuta como root
docker run --rm mi-app:1.0 whoami
# root (MAL) / appuser (BIEN)

# Verificar tamano
docker images mi-app
# Si es > 200MB, revisar si se puede optimizar

# Escanear vulnerabilidades
docker scout cves mi-app:1.0
```

> Los errores de construccion mas comunes tienen un patron comun: olvidar que una imagen de contenedor es un artefacto de produccion, no un entorno de desarrollo. Cada capa, cada paquete y cada configuracion impacta en la seguridad, el tamano y el rendimiento de la imagen.

