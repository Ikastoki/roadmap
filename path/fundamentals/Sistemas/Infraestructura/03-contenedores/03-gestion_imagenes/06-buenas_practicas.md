# Buenas practicas de imagenes

Las imagenes de contenedores son la unidad de despliegue. Una imagen bien construida es segura, eficiente en espacio, rapida de descargar y facil de mantener. Las buenas practicas de imagenes impactan directamente en la seguridad, el rendimiento y los costos de la operacion.

## Usar imagenes base minimas

Las imagenes base grandes (Ubuntu, Debian) incluyen cientos de paquetes que la aplicacion no necesita. Las imagenes minimas reducen la superficie de ataque y el tamano.

```dockerfile
# MAL: imagen base completa (~80MB)
FROM ubuntu:22.04

# BIEN: imagen minima (~5MB)
FROM alpine:3.19

# MEJOR: imagen distroless (sin shell, sin paquetes innecesarios)
FROM gcr.io/distroless/static-debian12

# MEJOR AUN: imagen chainguard (escaneada y actualizada continuamente)
FROM cgr.dev/chainguard/static:latest
```

## Limpiar cache en el mismo paso

Cada instruccion `RUN` crea una capa. Si se instalan paquetes y se limpia la cache en pasos separados, la cache queda en una capa anterior y el tamano no se reduce.

```dockerfile
# MAL: la cache de apt queda en una capa anterior
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3 python3-pip
RUN rm -rf /var/lib/apt/lists/*
# El tamano incluye ambos RUN

# BIEN: limpiar en el mismo paso
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip && \
    rm -rf /var/lib/apt/lists/*
# Un solo paso: la cache se limpia en la misma capa
```

## Ordenar instrucciones por frecuencia de cambio

Las instrucciones que cambian menos frecuente deben ir primero para maximizar el uso de la cache de Docker.

```dockerfile
# Orden optimizado:
FROM node:20-alpine                    # 1. Base (cambia raramente)
WORKDIR /app                          # 2. Directorio (nunca cambia)
COPY package.json package-lock.json . # 3. Dependencias (cambia poco)
RUN npm ci --production               # 4. Instalar dependencias
COPY src/ ./src/                      # 5. Codigo fuente (cambia frecuentemente)
COPY public/ ./public/                # 6. Recursos estaticos
EXPOSE 3000                           # 7. Metadatos
CMD ["node", "server.js"]             # 8. Comando por defecto
```

## Multi-stage builds

Construir en una imagen completa y copiar solo el artefacto final a una imagen minima de produccion.

```dockerfile
# Etapa 1: construccion
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Etapa 2: produccion
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]

# Resultado: imagen final sin dependencias de desarrollo
# Sin gcc, sin make, sin source maps
# Tamano: ~80MB en lugar de ~500MB
```

## Archivos de exclusion

```bash
# .dockerignore (similar a .gitignore)
node_modules
npm-debug.log
.git
.gitignore
Dockerfile
docker-compose.yml
.env
.env.*
*.md
.vscode
__pycache__
*.pyc
```

## No hardcodear secretos

```dockerfile
# PELIGROSO: secreto en la imagen
ENV API_KEY=sk-1234567890abcdef
# Cualquiera que haga inspect de la imagen vera la clave

# Seguro: pasar secretos en runtime
# docker run -e API_KEY=sk-1234567890abcdef mi-app
# O usar Docker/Kubernetes secrets
```

> Las buenas practicas de imagenes son inversiones en seguridad y eficiencia. Una imagen bien construida es mas pequena (descarga mas rapida), mas segura (menor superficie de ataque) y mas facil de mantener (construccion rapida y predecible).

