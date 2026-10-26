# Estructura del archivo de Compose

El archivo `docker-compose.yml` (o `compose.yml`) es un documento YAML que describe el estado deseado del sistema: servicios, redes, volumenes y configuraciones. Cada seccion tiene un proposito especifico y todas son opcionales excepto `services`.

## Estructura general

```yaml
# compose.yml
version: '3.8'

services:
  # Definicion de cada contenedor/servicio
  web:
    image: nginx:alpine
    # ... configuracion

  api:
    build: ./api
    # ... configuracion

networks:
  # Redes personalizadas
  frontend:
  backend:

volumes:
  # Volumenes nombrados
  db-data:

secrets:
  # Secretos sensibles
  db_password:
    file: ./secrets/db_password.txt
```

## Seccion services

```yaml
services:
  web:
    image: nginx:alpine           # Imagen base
    container_name: mi-web        # Nombre del contenedor
    restart: unless-stopped       # Politica de reinicio
    ports:
      - "8080:80"
    environment:
      NODE_ENV: production
    env_file:
      - .env
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - nginx-logs:/var/log/nginx
    networks:
      - frontend
    depends_on:
      - api
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
```

## Seccion networks

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Sin acceso a Internet
```

## Seccion volumes

```yaml
volumes:
  db-data:
    driver: local
  redis-data:
    driver: local
```

## Seccion secrets

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    environment: API_KEY
```

## Variables de entorno

```yaml
services:
  app:
    # Forma 1: lista de pares clave-valor
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - DB_PORT=5432

    # Forma 2: mapa de clave-valor
    environment:
      NODE_ENV: production
      DB_HOST: db

    # Forma 3: archivo .env
    env_file:
      - .env
      - .env.production
```

## Archivos .env

```bash
# .env
POSTGRES_DB=miapp
POSTGRES_USER=admin
POSTGRES_PASSWORD=secret123
REDIS_URL=redis://redis:6379
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
```

> La claridad del archivo de Compose es directamente proporcional a la mantenibilidad del sistema. Un archivo bien estructurado, con separacion clara de servicios, redes y volumenes, permite que cualquier miembro del equipo entienda y modifique el stack sin sorpresas.

