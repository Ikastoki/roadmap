# Desarrollo local con contenedores

Los contenedores eliminan el problema de "funciona en mi maquina" al empaquetar la aplicacion con todas sus dependencias. El desarrollo local con contenedores ofrece entornos reproducibles para todo el equipo, donde cada desarrollador ejecuta exactamente la misma configuracion sin necesidad de instalar servicios manualmente.

## El problema tradicional

```bash
# Sin contenedores: configuracion manual
sudo apt install postgresql redis-server nodejs npm
# Cada desarrollador tiene diferentes versiones
# "Funciona en mi maquina" es inevitable
```

## Solucion con Docker

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  app:
    build:
      context: .
      target: development
    ports:
      - "3000:3000"
    volumes:
      - ./src:/app/src  # Hot reload
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://dev:dev@db:5432/miapp_dev
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres-data:
```

```bash
# Un solo comando para todo el equipo
docker compose up -d
# PostgreSQL, Redis y la app estan listos
# Cada desarrollador tiene exactamente la misma configuracion
```

## Dev Containers

```json
// .devcontainer/devcontainer.json
{
  "name": "Mi App",
  "build": {
    "dockerfile": "Dockerfile.dev"
  },
  "features": {
    "ghcr.io/devcontainers/features/node:1": {},
    "ghcr.io/devcontainers/features/git:1": {}
  },
  "forwardPorts": [3000, 5432],
  "postCreateCommand": "npm install",
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode"
      ]
    }
  }
}
```

## Hot reload

```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app/src  # Codigo fuente montado
    # Cambios en ./src se reflejan inmediatamente en el contenedor
    # Sin reconstruir la imagen
```

## Ventajas del desarrollo con contenedores

- **Reproducibilidad:** todos ejecutan la misma configuracion.
- **Rapidez:** un solo comando para levantar todo el entorno.
- **Aislamiento:** no conflictos entre proyectos.
- **Portabilidad:** funciona en Linux, macOS y Windows.
- **Onboarding rapido:** nuevos desarrolladores productivos en minutos.

> El desarrollo local con contenedores transforma la configuracion de entornos de una tarea manual y propensa a errores en un proceso automatizado y reproducible. Un archivo `docker-compose.yml` es toda la documentacion necesaria para levantar un entorno completo de desarrollo.

