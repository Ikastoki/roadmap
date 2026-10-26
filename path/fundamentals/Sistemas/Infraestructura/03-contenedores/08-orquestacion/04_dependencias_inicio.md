# Dependencias y orden de inicio

El `depends_on` en Compose controla el orden de creacion de contenedores, pero no garantiza que un servicio este listo para recibir conexiones. Una base de datos puede tardar varios segundos en estar operativa despues de que su contenedor este en ejecucion. La solucion combina healthchecks con condiciones de dependencia.

## El problema

```yaml
services:
  db:
    image: postgres:16-alpine

  app:
    image: node:20
    depends_on:
      - db
    # App inicia DESPUES de que db se crea, pero...
    # PostgreSQL tarda 5-10 segundos en estar listo
    # La app falla con "connection refused"
```

## Solucion: healthcheck + condition

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d miapp"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    image: node:20
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    # App solo inicia cuando db y redis estan saludables
```

## Opciones de condition

```yaml
depends_on:
  db:
    condition: service_started      # Solo necesita que el contenedor este en ejecucion
  redis:
    condition: service_healthy      # Necesita que el healthcheck pase
  worker:
    condition: service_completed_successfully  # Necesita que termine exitosamente
```

## Scripts de espera alternativos

```bash
# wait-for-it.sh (descargable)
# Espera a que un host:puerto este disponible
./wait-for-it.sh db:5432 --timeout=30 --strict -- echo "db is up"

# En Dockerfile
COPY wait-for-it.sh /usr/local/bin/
CMD ["wait-for-it.sh", "db:5432", "--", "node", "server.js"]
```

```yaml
services:
  app:
    image: node:20
    command: >
      sh -c "wait-for-it.sh db:5432 -- node server.js"
    depends_on:
      - db
```

## Patron completo

```yaml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d miapp"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    secrets:
      - db_password

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build: ./api
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://admin@db:5432/miapp
      REDIS_URL: redis://redis:6379
    ports:
      - "3000:3000"

volumes:
  db-data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

> El `depends_on` sin `condition` es una trampa comun: garantiza el orden de creacion pero no la disponibilidad del servicio. La combinacion de healthchecks y `condition: service_healthy` es la forma correcta de gestionar dependencias en Compose, y debe ser la norma, no la excepcion.

