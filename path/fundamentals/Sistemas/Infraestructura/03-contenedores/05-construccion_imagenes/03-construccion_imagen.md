# Construccion de la imagen

El comando `docker build` toma un Dockerfile y un contexto de construccion, ejecuta las instrucciones secuencialmente y genera una imagen de contenedor. Cada instruccion exitosa crea una capa en la imagen, y las capas se almacenan en cache para acelerar construcciones futuras.

## Comando de construccion

```bash
# Construccion basica
docker build -t mi-app:1.0 .

# Contexto: el directorio actual (.)
# -t: etiqueta (nombre:tag)
# El punto (.) indica el contexto de construccion

# Construir desde un Dockerfile especifico
docker build -f Dockerfile.prod -t mi-app:prod .

# Construir con argumentos
docker build --build-arg NODE_VERSION=18 -t mi-app:1.0 .

# Construir sin cache
docker build --no-cache -t mi-app:1.0 .

# Construir y mostrar output detallado
docker build --progress=plain -t mi-app:1.0 .
```

## Contexto de construccion

El contexto es el directorio que se envia al motor de contenedores. Todos los archivos y directorios dentro del contexto estan disponibles para las instrucciones COPY y ADD.

```bash
# Construir con contexto de un directorio especifico
docker build -t mi-app:1.0 /ruta/al/proyecto

# El Dockerfile puede estar en otro lugar
docker build -f /otra/ruta/Dockerfile -t mi-app:1.0 /ruta/al/proyecto
```

**.dockerignore:** excluye archivos del contexto.

```bash
# .dockerignore
.git
node_modules
npm-debug.log
.env
.env.*
*.md
docker-compose.yml
.vscode
__pycache__
*.pyc
dist/
build/
```

## Proceso de construccion

```
Paso 1/8: FROM node:20-alpine
 ---> a1b2c3d4e5f6
Paso 2/8: WORKDIR /app
 ---> Ejecutando en abc123... dentro de la capa a1b2c3d4e5f6
 ---> b2c3d4e5f6a1
Paso 3/8: COPY package*.json ./
 ---> c3d4e5f6a1b2
Paso 4/8: RUN npm ci --production
 ---> Ejecutando en d4e5f6a1b2c3 dentro de la capa c3d4e5f6a1b2
 ---> e5f6a1b2c3d4
...
Successfully built e5f6a1b2c3d4
Successfully tagged mi-app:1.0
```

## Cache de capas

Docker almacena las capas y las reutiliza en construcciones futuras. Si una capa y sus dependencias no cambiaron, Docker la omite y usa la version en cache.

```
Con cache:
Paso 1/4: FROM node:20-alpine → cache hit (0.1s)
Paso 2/4: WORKDIR /app → cache hit (0.1s)
Paso 3/4: COPY package*.json ./ → cache hit (0.2s)
Paso 4/4: RUN npm ci → cache hit (0.5s)
Total: 0.9s (construccion cacheada)

Sin cache:
Paso 1/4: FROM node:20-alpine → descargando... (10s)
Paso 2/4: WORKDIR /app → ejecutando... (1s)
Paso 3/4: COPY package*.json ./ → copiando... (0.5s)
Paso 4/4: RUN npm ci → instalando... (60s)
Total: 71.5s (primera construccion)
```

## Verificar la imagen construida

```bash
# Ver la imagen
docker images mi-app
# REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
# mi-app       1.0       e5f6a1b2c3d4   5 seconds ago    150MB

# Inspeccionar
docker inspect mi-app:1.0

# Ejecutar para probar
docker run --rm -p 3000:3000 mi-app:1.0

# Analizar con dive
dive mi-app:1.0
```

## Errores comunes de construccion

```bash
# Error: "COPY failed: file not found"
# Causa: archivo no existe en el contexto
# Solucion: verificar que el archivo existe y esta en .dockerignore

# Error: "npm ERR! code E404"
# Causa: package.json no se copio correctamente
# Solucion: verificar el orden de instrucciones COPY

# Error: "permission denied"
# Causa: el contenedor intenta escribir como root
# Solucion: usar USER antes de COPY con --chown
```

> La construccion de imagenes es un proceso determinista: el mismo Dockerfile con el mismo contexto siempre genera la misma imagen. Esta reproducibilidad es lo que hace a los contenedores confiables para produccion.

