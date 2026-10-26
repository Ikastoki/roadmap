# Publicacion de imagenes

El flujo de publicacion de imagenes sigue tres pasos: etiquetar la imagen con el nombre del registry, autenticarse en el registry y subir la imagen. Este proceso convierte una imagen local en una imagen distribuible que puede ser descargada por otros desarrolladores o servidores.

## Flujo basico

```bash
# 1. Etiquetar la imagen
docker tag mi-app:latest docker.io/miusuario/mi-app:1.0

# 2. Autenticarse
docker login
# Username: miusuario
# Password: ****
# Login Succeeded

# 3. Subir
docker push docker.io/miusuario/mi-app:1.0
# The push refers to repository [docker.io/miusuario/mi-app]
# abc123: Pushed
# def456: Pushed
# 1.0: digest: sha256:abc123... size: 1234
```

## Push con build automatico

```bash
# Build y push en un solo comando
docker buildx build --push -t docker.io/miusuario/mi-app:1.0 .

# Build multiplataforma y push
docker buildx build --push --platform linux/amd64,linux/arm64 \
  -t docker.io/miusuario/mi-app:1.0 .
```

## Login con tokens

```bash
# Docker Hub
docker login
# Credenciales en ~/.docker/config.json

# AWS ECR (token temporal)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Google Cloud
gcloud auth print-access-token | docker login -u oauth2accesstoken --password-stdin https://us-central1-docker.pkg.dev
```

## Desautenticarse

```bash
# Cerrar sesion
docker logout
# Removing login credentials for https://index.docker.io/v1/

# Cerrar sesion de un registry especifico
docker logout micr.azurecr.io
```

## Buena practica de etiquetado

```bash
# Semantic Versioning
docker tag mi-app:latest miusuario/mi-app:v1.0.0
docker tag mi-app:latest miusuario/mi-app:v1.0.1
docker tag mi-app:latest miusuario/mi-app:v1.1.0

# Con SHA del commit
docker tag mi-app:latest miusuario/mi-app:sha-abc1234

# latest solo para desarrollo, nunca para produccion
docker tag mi-app:latest miusuario/mi-app:latest  # Solo dev
```

> El flujo de publicacion es el puente entre el desarrollo local y la distribucion global. Etiquetar, autenticar y subir son tres pasos simples, pero la discipline en el etiquetado (versiones, SHAs, sin `latest` en produccion) es lo que marca la diferencia entre un repositorio ordenado y uno caotico.

