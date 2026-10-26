# Integracion continua

La integracion continua con contenedores ejecuta suites de pruebas en entornos efimeros y limpios. Cada ejecucion del pipeline crea contenedores temporales, ejecuta las pruebas y elimina todo, garantizando que las pruebas no dependan de un estado previo.

## Patron basico

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install
        run: npm ci

      - name: Run tests
        run: npm test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
```

## Contenedores efimeros para pruebas

```yaml
# Cada test suite corre en su propio contenedor
services:
  unit-tests:
    build:
      context: .
      target: test
    command: npm run test:unit

  integration-tests:
    build:
      context: .
      target: test
    command: npm run test:integration
    depends_on:
      db:
        condition: service_healthy
```

## Build y test en paralelo

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: docker build -t mi-app:test .

  test-unit:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Run unit tests
        run: docker run --rm mi-app:test npm run test:unit

  test-integration:
    needs: build
    runs-on: ubuntu-latest
    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
    steps:
      - name: Run integration tests
        run: docker run --rm --network host mi-app:test npm run test:integration
```

## Multi-stage para testing

```dockerfile
# Dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS development
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS test
COPY . .
RUN npm run build
CMD ["npm", "test"]

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=test /app/dist ./dist
COPY --from=test /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

> La integracion continua con contenedores garantiza que cada cambio se prueba en un entorno limpio y reproducible. Los servicios de base de datos y cache se levantan como contenedores efimeros en el pipeline, eliminando la dependencia de configuraciones manuales y asegurando que las pruebas sean consistentes en cada ejecucion.

