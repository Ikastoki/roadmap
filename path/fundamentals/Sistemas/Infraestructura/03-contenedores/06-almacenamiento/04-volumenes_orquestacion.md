# Volumenes en orquestacion local

En orquestadores locales como Docker Compose, los volumenes se definen de forma declarativa en el archivo de composicion. Esto permite que los datos persistan entre ejecuciones de `docker compose up` y `docker compose down`, y que multiples servicios compartan los mismos volumenes.

## Definicion en Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro  # Bind mount

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data

  app:
    image: node:20-alpine
    volumes:
      - ./src:/app/src  # Bind mount para desarrollo
    depends_on:
      - db
      - redis

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
```

## Comportamiento con `docker compose down`

```bash
# down sin --volumes: conserva los volumenes
docker compose down
# Contenedores eliminados, volumenes intactos

# down con --volumes: elimina volumenes anonimos
docker compose down -v
# Contenedores Y volumenes anonimos eliminados
# Los volumenes nombrados declarados en `volumes:` NO se eliminan
```

## Volumenes nombrados vs. anonimos

```bash
# Volumen nombrado (persiste con down)
volumes:
  - postgres-data:/var/lib/postgresql/data

# Volumen anonimo (se elimina con down -v)
volumes:
  - /var/lib/postgresql/data
```

## Compartir volumenes entre servicios

```yaml
version: '3.8'
services:
  app1:
    image: my-app
    volumes:
      - shared-data:/data

  app2:
    image: my-app
    volumes:
      - shared-data:/data

volumes:
  shared-data:
# Ambos servicios comparten el mismo volumen
```

## Drivers de volumen

```yaml
volumes:
  datos-remotos:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.200,rw,nfsvers=4
      device: ":/srv/nfs/datos"
```

## Buena practica

```yaml
# Estructura recomendada
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

volumes:
  db-data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

> Los volumenes declarativos en Compose son la base del almacenamiento persistente en entornos de desarrollo y pruebas. La combinacion de volumenes nombrados para datos persistentes y bind mounts para codigo en desarrollo crea un entorno que es tanto reproducible como productivo.

