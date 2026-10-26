# Registro de imagenes

Un registry es un servicio centralizado que almacena, versiona y distribuye imagenes de contenedores. Docker Hub es el registry publico por defecto, pero las organizaciones suelen usar registries privados para controlar el acceso, la seguridad y la distribucion de sus imagenes.

## Docker Hub

```bash
# Buscar imagenes
docker search nginx
# NAME            DESCRIPTION               STARS   OFFICIAL
# nginx           Official build of Nginx   15000   [OK]

# Pull de imagenes
docker pull nginx:alpine
docker pull postgres:16-alpine
docker pull node:20-alpine

# Ver imagenes locales
docker images
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# nginx        alpine    abc123         2 hours ago    42MB
# postgres     16-alpine def456         3 hours ago    230MB
```

## Registries privados

```bash
# Registry basico (Docker Registry)
docker run -d -p 5000:5000 --name registry registry:2

# Etiquetar y push
docker tag mi-app:latest localhost:5000/mi-app:1.0
docker push localhost:5000/mi-app:1.0

# Pull desde registry privado
docker pull localhost:5000/mi-app:1.0
```

## Cloud registries

```bash
# AWS ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
docker tag mi-app:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/mi-app:1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/mi-app:1.0

# Google Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev
docker tag mi-app:latest us-central1-docker.pkg.dev/proyecto/mi-repo/mi-app:1.0
docker push us-central1-docker.pkg.dev/proyecto/mi-repo/mi-app:1.0

# Azure Container Registry
az acr login --name micr
docker tag mi-app:latest micr.azurecr.io/mi-app:1.0
docker push micr.azurecr.io/mi-app:1.0
```

## Etiquetado

```bash
# Formato: registry/usuario/repositorio:tag
docker tag mi-app:latest docker.io/miusuario/mi-app:1.0
docker tag mi-app:latest docker.io/miusuario/mi-app:latest
docker tag mi-app:latest docker.io/miusuario/mi-app:sha-abc1234
```

> Los registries son la columna vertebral de la distribucion de imagenes. Docker Hub es el punto de partida para imagenes publicas, pero los registries privados son esenciales para el control de acceso, la seguridad y la cumplimentacion en entornos empresariales.

