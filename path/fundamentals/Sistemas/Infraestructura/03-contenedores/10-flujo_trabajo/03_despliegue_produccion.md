# Despliegue en produccion

El despliegue de contenedores en produccion tiene dos caminos: ejecucion directa en motores de contenedores para aplicaciones simples, y plataformas orquestadas para aplicaciones distribuidas y microservicios. La eleccion depende de la complejidad, los requisitos de disponibilidad y la escala.

## Ejecucion directa

```bash
# Despliegue simple con Docker
docker pull miusuario/mi-app:v1.2.3
docker run -d \
  --name mi-app \
  --restart unless-stopped \
  -p 8080:3000 \
  -v app-data:/app/data \
  --memory 512m \
  --cpus 1 \
  miusuario/mi-app:v1.2.3
```

```yaml
# docker-compose.yml en el servidor
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
    restart: unless-stopped

  app:
    image: miusuario/mi-app:v1.2.3
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
    volumes:
      - app-data:/app/data
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres-data:
  app-data:
```

## Plataformas orquestadas

```bash
# Kubernetes: despliegue con un solo comando
kubectl apply -f deployment.yml

# Amazon ECS
aws ecs update-service --cluster mi-cluster --service mi-app --force-new-deployment

# HashiCorp Nomad
nomad job run mi-app.nomad
```

## Actualizaciones con cero downtime

```bash
# Docker Compose: recrear sin detener
docker compose up -d --no-deps --build app
# Recrea solo el contenedor app sin afectar otros servicios

# Rolling update con Docker Swarm
docker service update --image miusuario/mi-app:v1.2.3 mi-app
```

## Rollback

```bash
# Docker Compose: volver a version anterior
# Editar compose.yml con la imagen anterior
docker compose up -d app

# Kubernetes
kubectl rollout undo deployment/mi-app
```

## Monitoreo basico

```bash
# Ver estado de contenedores
docker compose ps

# Ver recursos
docker stats

# Ver logs en produccion
docker compose logs --tail 100 -f app
```

> El despliegue de contenedores en produccion escala desde ejecucion directa para aplicaciones simples hasta plataformas orquestadas para sistemas distribuidos. Para la mayoria de aplicaciones, la combinacion de Docker Compose con una buena estrategia de imagenes versionadas y actualizaciones con cero downtime es suficiente.

