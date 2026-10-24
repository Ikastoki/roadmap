# Docker en el desarrollo local

El desarrollo de software moderno exige entornos que sean consistentes entre los miembros del equipo, independientemente del sistema operativo o la configuracion de cada maquina. Docker resuelve este problema empaquetando la aplicacion y todas sus dependencias en una imagen que cualquier desarrollador puede ejecutar con un solo comando. Ya no existen discrepancias entre "funciona en mi maquina" porque la maquina es siempre el mismo contenedor.

## Entornos reproducibles con Docker Compose

Un proyecto tipico involucra multiples servicios: una base de datos, un cache, una cola de mensajes y la aplicacion misma. Docker Compose permite definir toda la topologia en un archivo YAML y levantarla con un solo comando.

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: devpassword
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

Cada desarrollador ejecuta `docker compose up` y obtiene exactamente el mismo entorno. La base de datos se inicializa con la misma version de Postgres, el cache utiliza la misma version de Redis y la aplicacion se compila con las mismas dependencias. No importa si el host es macOS, Windows o Linux: el interior del contenedor es identico.

> Para acelerar los arranques iniciales, es practica habitual precargar las imagenes base en un registro local o utilizar mirrors de Docker Hub. En equipos grandes, un registro privado reduce drasticamente el tiempo de incorporacion de nuevos desarrolladores.

## Volumenes y montajes bind

Los volumenes son el mecanismo de Docker para persistir datos y compartirlos entre el host y el contenedor. Existen dos variantes principales: los volumenes gestionados por Docker y los bind mounts.

Los bind mounts mapean un directorio del host directamente dentro del contenedor. Son la opcion preferida en desarrollo porque permiten editar el codigo en el host y ver los cambios reflejados instantaneamente en el contenedor sin reconstruir la imagen.

```bash
# Montar el directorio actual como /app dentro del contenedor
docker run -v "$(pwd):/app" node:20 npm run dev
```

Los volumenes gestionados, por otro lado, se almacenan en el area de datos de Docker (`/var/lib/docker/volumes/`) y son la opcion recomendada para produccion porque ofrecen mejor rendimiento y son portables entre hosts.

```bash
# Crear y usar un volumen gestionado
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres:16
```

## Hot reload en contenedores

El desarrollo local requiere ciclos de retroalimentacion rapidos. La combinacion de bind mounts con herramientas de recarga automatica permite que el contenedor se reconstruya o reinicie cuando los archivos cambian.

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npx", "nodemon", "src/index.js"]
```

Con el bind mount activo, cualquier cambio en el directorio del host desencadena un reinicio del proceso dentro del contenedor. Herramientas como `nodemon`, `air` (Go), `spring-boot-devtools` (Java) o `uvicorn --reload` (Python) funcionan perfectamente en este esquema.

Para proyectos que requieren compilacion (TypeScript, Rust, Go), se puede anadir un paso de `watch`:

```json
{
  "scripts": {
    "dev:ts": "tsc --watch & nodemon dist/index.js"
  }
}
```

```bash
docker compose up app  # nodemon vigila los cambios
```

> Es importante excluir directorios como `node_modules` del bind mount para evitar problemas de rendimiento y compatibilidad. La tecnica del `node_modules` anonimo (`/app/node_modules`) monta un volumen temporal que oculta el directorio del host y permite que las dependencias nativas se compilen dentro del contenedor.

## Perfiles de Docker Compose

No todos los desarrolladores necesitan todos los servicios todo el tiempo. Docker Compose permite definir perfiles para activar o desactivar servicios segun la tarea:

```yaml
services:
  app:
    image: myapp:dev
    profiles: ["core"]

  celery-worker:
    image: myapp:dev
    profiles: ["workers"]

  mailhog:
    image: mailhog/mailhog
    profiles: ["mail", "core"]
```

```bash
# Solo la aplicacion
docker compose --profile core up

# Aplicacion con workers
docker compose --profile core --profile workers up
```

Esta flexibilidad evita consumir recursos innecesarios y acelera el arranque del entorno de desarrollo.

## Variables de entorno y secretos locales

La configuracion del entorno debe ser explicita y por servicio. Docker Compose soporta archivos `.env` para centralizar las variables:

```bash
# .env
POSTGRES_PASSWORD=devpassword
REDIS_PASSWORD=devpassword
API_URL=http://localhost:3000
```

```yaml
services:
  app:
    env_file:
      - .env
```

Para secretos sensibles que no deben versionarse, se utiliza un archivo `.env.local` ignorado por git que cada desarrollador gestiona individualmente. En produccion estos secretos se inyectan mediante Docker secrets o Vault.

## Gestion de multiples entornos

Un mismo proyecto puede definir varios archivos Compose para diferentes escenarios:

- `compose.yaml` - configuracion base comun
- `compose.override.yaml` - se carga automaticamente y anade configuracion local
- `compose.prod.yaml` - configuracion de produccion (sin bind mounts, con reinicios)
- `compose.test.yaml` - configuracion para tests (base de datos en memoria, puertos fijos)

```bash
# Desarrollo
docker compose up

# Tests
docker compose -f compose.yaml -f compose.test.yaml up --abort-on-container-exit
```

Este patron permite compartir una base comun y especializar el comportamiento sin duplicar configuracion.

