# Variables de entorno y archivos .env

La separacion de la configuracion del codigo es un principio fundamental del desarrollo de software. Docker Compose facilita esta separacion mediante variables de entorno que se inyectan en los contenedores, permitiendo la misma imagen en diferentes entornos (desarrollo, staging, produccion) con diferentes configuraciones.

## Formas de definir variables

```yaml
services:
  app:
    image: node:20

    # Forma 1: lista de pares
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - DB_PORT=5432

    # Forma 2: mapa clave-valor
    environment:
      NODE_ENV: production
      DB_HOST: db

    # Forma 3: archivo .env
    env_file:
      - .env
      - .env.production
```

## Archivo .env

```bash
# .env (en el mismo directorio que compose.yml)
# Este archivo NO debe subirse a repositorios (agregar a .gitignore)
POSTGRES_DB=miapp
POSTGRES_USER=admin
POSTGRES_PASSWORD=supersecret123
REDIS_URL=redis://redis:6379
API_KEY=abc123def456
```

```yaml
# compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

  app:
    image: node:20
    env_file:
      - .env
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      REDIS_URL: ${REDIS_URL}
```

## Multiples entornos

```bash
# .env (desarrollo por defecto)
POSTGRES_DB=miapp_dev
POSTGRES_USER=dev_user
POSTGRES_PASSWORD=dev_pass
NODE_ENV=development

# .env.production
POSTGRES_DB=miapp
POSTGRES_USER=prod_user
POSTGRES_PASSWORD=prod_verysecret
NODE_ENV=production
```

```bash
# Usar archivo .env especifico
docker compose --env-file .env.production up -d
```

## Variables obligatorias y por defecto

```bash
# Variables obligatorias (faltan = error)
services:
  app:
    environment:
      - API_KEY=${API_KEY}  # Error si API_KEY no esta definida

# Variables con valor por defecto
services:
  app:
    environment:
      - NODE_ENV=${NODE_ENV:-development}
      - PORT=${PORT:-3000}
      - DB_HOST=${DB_HOST:-localhost}
```

## Variables del sistema

```yaml
services:
  app:
    image: node:20
    environment:
      # Variables del host
      - HOME=${HOME}
      - USER=${USER}
      # Variables especificas de Docker
      - CONTAINER_NAME=mi-app
```

## Secrets para datos criticos

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

## Buenas practicas

```bash
# .gitignore
.env
.env.local
.env.production

# Ejemplo de .env.example (si se sube al repo)
POSTGRES_DB=miapp
POSTGRES_USER=admin
POSTGRES_PASSWORD=cambiar_este_valor
REDIS_URL=redis://redis:6379
```

> Las variables de entorno son el mecanismo estandar para la configuracion de aplicaciones en contenedores. La combinacion de archivos `.env` para valores por defecto, `--env-file` para entornos especificos y secrets para datos criticos crea un sistema de configuracion que es seguro, flexible y mantenible.

