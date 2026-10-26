# Intro a Docker Compose

Docker Compose es una herramienta para definir y ejecutar aplicaciones multi-contenedor de forma declarativa. En lugar de ejecutar multiples comandos `docker run`, un unico archivo YAML describe todos los servicios, redes, volumenes y configuraciones. Con un solo comando se levanta todo el stack.

## El problema que resuelve

```bash
# Sin Compose: multiples comandos manuales
docker network create mi-app
docker volume create postgres-data
docker run -d --name db --network mi-app -v postgres-data:/var/lib/postgresql/data postgres:16
docker run -d --name redis --network mi-app redis:7
docker run -d --name app --network mi-app -p 3000:3000 mi-app
# 5 comandos, dificil de reproducir, facil de olvidar un paso
```

```yaml
# Con Compose: un archivo declarativo
version: '3.8'
services:
  db:
    image: postgres:16-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

  app:
    image: mi-app
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis

volumes:
  postgres-data:
```

```bash
# Un solo comando
docker compose up -d
# Crea red, volumen y todos los servicios automaticamente
```

## Versiones

```bash
# Docker Compose v2 (actual, integrado en Docker)
docker compose up -d
docker compose down

# Docker Compose v1 (obsoleto, comando separado)
docker-compose up -d
docker-compose down
```

## Comandos基本icos

```bash
# Iniciar servicios en segundo plano
docker compose up -d

# Ver servicios activos
docker compose ps

# Ver logs
docker compose logs -f

# Detener servicios (conserva contenedores)
docker compose stop

# Detener y eliminar contenedores
docker compose down

# Detener y eliminar contenedores + volumenes
docker compose down -v

# Reconstruir imagenes y recrear contenedores
docker compose up -d --build

# Ejecutar comando en un servicio
docker compose exec db psql -U admin
```

## Dependencias entre servicios

```yaml
version: '3.8'
services:
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    image: node:20
    depends_on:
      db:
        condition: service_healthy
    # App solo inicia cuando db esta saludable
```

> Docker Compose transforma la gestion de multiples contenedores de una tarea manual y propensa a errores en un proceso declarativo y reproducible. Un solo archivo describe el estado deseado del sistema, y Compose se encarga de crear, configurar y conectar todos los componentes automaticamente.

