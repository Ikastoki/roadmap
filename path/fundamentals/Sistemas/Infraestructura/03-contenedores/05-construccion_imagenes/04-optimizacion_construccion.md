# Optimizacion de la construccion

Una imagen bien optimizada es mas pequena (descarga mas rapida, consume menos espacio), mas segura (menor superficie de ataque) y se construye mas rapido (mejor uso de cache). La optimizacion no es un paso opcional: es una practica obligatoria para imagenes de produccion.

## Ordenar por frecuencia de cambio

```dockerfile
# MAL: el codigo cambia frecuentemente, las dependencias no
COPY src/ ./src/
COPY package.json ./
RUN npm ci
# Si src/ cambia, npm ci se ejecuta de nuevo (cache invalidada)

# BIEN: dependencias primero, codigo despues
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
# Si solo cambia src/, npm ci se reutiliza del cache
```

## Combinar instrucciones RUN

```dockerfile
# MAL: 3 capas
RUN apt-get update
RUN apt-get install -y python3
RUN rm -rf /var/lib/apt/lists/*
# La cache de apt queda en la capa 1, no se puede limpiar

# BIEN: 1 capa
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 && \
    rm -rf /var/lib/apt/lists/*
# Todo en una capa: se puede limpiar la cache
```

## Multi-stage builds

```dockerfile
# Etapa de construccion
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Etapa de produccion
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]

# Resultado: imagen sin gcc, make, source maps, tests
# Tamano: ~80MB vs ~500MB
```

```dockerfile
# Python multi-stage
FROM python:3.12 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

## Imagenes base minimas

```dockerfile
# MAL: Ubuntu completo (~80MB base + paquetes)
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3

# BIEN: Python slim (~50MB)
FROM python:3.12-slim

# MEJOR: Distroless (~20MB, sin shell)
FROM gcr.io/distroless/python3-debian12

# MINIMO: Alpine (~5MB, pero con musl en lugar de glibc)
FROM python:3.12-alpine
```

## Usar .dockerignore efectivamente

```dockerignore
.git
.github
.vscode
.idea
node_modules
npm-debug.log
.env
.env.*
*.md
docker-compose*.yml
Dockerfile*
.dockerignore
dist/
build/
coverage/
tests/
__tests__/
*.test.js
*.spec.js
```

## Verificar tamano

```bash
# Ver tamano de la imagen
docker images mi-app
# REPOSITORY   TAG       SIZE
# mi-app       1.0       150MB

# Ver tamano de cada capa
docker history mi-app:1.0

# Analizar eficiencia con dive
dive mi-app:1.0 --ci
# Image efficiency score: 92.3%
# Size: 150MB
# Optimizable: 12MB
```

> La optimizacion de imagenes es un proceso continuo: cada nueva version del Dockerfile debe ser revisada para mantener la eficiencia. Las multi-stage builds, las imagenes base minimas y el uso efectivo de cache son las herramientas que convierten una imagen de 500MB en una de 80MB sin perder funcionalidad.

