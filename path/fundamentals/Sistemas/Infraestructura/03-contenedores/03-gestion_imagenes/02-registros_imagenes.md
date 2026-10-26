# Registros de imagenes (Registries)

Un registry es un servicio centralizado que almacena, versiona y distribuye imagenes de contenedores. Es el equivalente a un repositorio de Git pero para imagenes: los desarrolladores suben (push) imagenes construidas y las aplicaciones o servidores las descargan (pull) cuando necesitan ejecutar contenedores.

## Registros publicos

**Docker Hub (hub.docker.com):**
El registro publico mas grande y utilizado. Contiene millones de imagenes oficiales y comunitarias.

```bash
# Buscar imagenes en Docker Hub
docker search redis
# NAME        DESCRIPTION               STARS   OFFICIAL
# redis       Redis is an in-memory...  12000   [OK]

# Descargar imagen oficial
docker pull redis:7-alpine

# Descargar imagen comunitaria
docker pull bitnami/redis:7.2
```

**Quay.io (Red Hat):**
Registro publico y privado con foco en seguridad y escaneo de vulnerabilidades.

```bash
docker pull quay.io/prometheus/prometheus:latest
```

**GitHub Container Registry (ghcr.io):**
Integrado con GitHub. Las imagenes se asocian a repositorios de GitHub.

```bash
docker pull ghcr.io/usuario/proyecto:1.0
```

**Google Container Registry (gcr.io):**
Registro de Google Cloud para imagenes de Google Cloud y propias.

```bash
docker pull gcr.io/distroless/static-debian12
```

## Registros privados empresariales

Las organizaciones que no quieren subir sus imagenes a registros publicos utilizan registros privados.

**Docker Registry (self-hosted):**

```bash
# Ejecutar registro privado
docker run -d -p 5000:5000 --name registry \
  -v registry-data:/var/lib/registry \
  registry:2

# Etiquetar imagen para el registro privado
docker tag mi-app:1.0 localhost:5000/mi-app:1.0

# Subir imagen
docker push localhost:5000/mi-app:1.0
```

**Harbor:**
Registro empresarial open-source con escaneo de vulnerabilidades, control de acceso y auditoria.

```bash
# Harbor se despliega con Docker Compose o Helm
# Proporciona interfaz web, RBAC, escaneo automatico
# y replicacion entre registros
```

**Servicios gestionados en la nube:**

```bash
# AWS ECR (Elastic Container Registry)
aws ecr create-repository --repository-name mi-app
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 123456789.dkr.ecr.eu-west-1.amazonaws.com
docker tag mi-app:1.0 123456789.dkr.ecr.eu-west-1.amazonaws.com/mi-app:1.0
docker push 123456789.dkr.ecr.eu-west-1.amazonaws.com/mi-app:1.0

# Azure Container Registry (ACR)
az acr create --name milegistro --sku Basic
az acr login --name milegistro
docker tag mi-app:1.0 milegistro.azurecr.io/mi-app:1.0
docker push milegistro.azurecr.io/mi-app:1.0

# Google Artifact Registry
gcloud artifacts repositories create mi-repo --repository-format=docker --location=europe-west1
docker tag mi-app:1.0 europe-west1-docker.pkg.dev/proyecto/mi-repo/mi-app:1.0
docker push europe-west1-docker.pkg.dev/proyecto/mi-repo/mi-app:1.0
```

## Autenticacion

```bash
# Login en Docker Hub
docker login
# Username: mi-usuario
# Password: mi-token (no la contrasena)

# Login en registro privado
docker login registry.empresa.com
# Username: admin
# Password: token-seguro

# Login en AWS ECR
aws ecr get-login-password | docker login --username AWS --password-stdin 123456789.dkr.ecr.eu-west-1.amazonaws.com
```

> Los registros de imagenes son la columna vertebral de la distribucion de contenedores. Docker Hub es el punto de partida para imagenes publicas; los registros privados y gestionados son esenciales para organizaciones que necesitan control de acceso, seguridad y cumplimiento normativo.

