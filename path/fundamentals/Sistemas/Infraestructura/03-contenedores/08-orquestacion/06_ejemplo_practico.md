# Ejemplo practico: stack completo

Un stack completo en Docker Compose demuestra como multiples servicios se conectan y comunican: una aplicacion web, una base de datos, un proxy inverso y un servicio de cache. Cada componente tiene un rol especifico y se comunica a traves de redes y volumenes declarativos.

## Arquitectura

```
Internet ──► nginx (proxy) ──► app (node) ──► db (postgres)
                                     │
                                     └──► redis (cache)
```

## compose.yml

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/usr/share/nginx/html:ro
    depends_on:
      - app
    networks:
      - frontend
    restart: unless-stopped

  app:
    build: ./app
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://admin:${POSTGRES_PASSWORD}@db:5432/miapp
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d miapp"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

networks:
  frontend:
  backend:
    internal: true

volumes:
  postgres-data:
  redis-data:
```

## .env

```bash
POSTGRES_PASSWORD=mi_password_seguro
REDIS_PASSWORD=redis_seguro
```

## nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /static {
            alias /usr/share/nginx/html;
        }
    }
}
```

## Despliegue

```bash
# Crear directorios
mkdir -p app nginx html secrets

# Crear .env
echo "POSTGRES_PASSWORD=$(openssl rand -hex 16)" > .env
echo "REDIS_PASSWORD=$(openssl rand -hex 16)" >> .env

# Crear secretos
echo "mi_password_seguro" > secrets/db_password.txt

# Desplegar
docker compose up -d --build

# Verificar
docker compose ps
docker compose logs -f app
```

> Un stack completo en Compose es la prueba de que la orquestacion local puede manejar aplicaciones reales con multiples servicios, redes aisladas, volumenes persistentes y dependencias saludables. Cada servicio tiene una responsabilidad clara y se comunica a traves de una red definida, creando un sistema que es tanto funcional como mantenible.

