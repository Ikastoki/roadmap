# Registros privados y empresariales

Los registries privados ofrecen control total sobre el acceso, la seguridad y la distribucion de imagenes. Las organizaciones los usan para cumplir politicas de seguridad, gestionar imagenes propietarias y reducir la dependencia de registries publicos.

## Docker Registry (basico)

```bash
# Ejecutar registry privado
docker run -d -p 5000:5000 \
  --name registry \
  -v registry-data:/var/lib/registry \
  registry:2

# Etiquetar y push
docker tag mi-app:latest localhost:5000/mi-app:1.0
docker push localhost:5000/mi-app:1.0

# Pull
docker pull localhost:5000/mi-app:1.0
```

```yaml
# En Docker Compose
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry

volumes:
  registry-data:
```

## Harbor (empresarial open source)

```yaml
# Instalacion con Docker Compose
# Descargar Harbor: https://goharbor.io
# Configurar harbor.yml

# Caracteristicas:
# - UI web para gestion de imagenes
# - Escaneo de vulnerabilidades integrado
# - Control de acceso basado en proyectos
# - Replicacion entre registries
# - Firmado de imagenes
```

## Cloud registries gestionados

```bash
# AWS ECR (Elastic Container Registry)
# Crear repositorio
aws ecr create-repository --repository-name mi-app --region us-east-1

# Login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Push
docker tag mi-app:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/mi-app:1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/mi-app:1.0

# Politica de limpieza: eliminar imagenes antiguas
aws ecr put-lifecycle-policy \
  --repository-name mi-app \
  --lifecycle-policy-text '{"rules":[{"rulePriority":1,"selection":{"tagStatus":"untagged","countType":"imageCountMoreThan","countNumber":10},"action":{"type":"expire"}}]}'
```

```bash
# Google Artifact Registry
# Crear repositorio
gcloud artifacts repositories create mi-repo --repository-format=docker --location=us-central1

# Login
gcloud auth configure-docker us-central1-docker.pkg.dev

# Push
docker tag mi-app:latest us-central1-docker.pkg.dev/proyecto/mi-repo/mi-app:1.0
docker push us-central1-docker.pkg.dev/proyecto/mi-repo/mi-app:1.0
```

```bash
# Azure Container Registry
# Crear registro
az acr create --resource-group mi-rg --name micr --sku Basic

# Login
az acr login --name micr

# Push
docker tag mi-app:latest micr.azurecr.io/mi-app:1.0
docker push micr.azurecr.io/mi-app:1.0
```

## Comparacion

| Caracteristica | Docker Registry | Harbor | ECR | Artifact Registry | ACR |
|---|---|---|---|---|---|
| Hospedaje | Self-hosted | Self-hosted | AWS | Google | Azure |
| Escaneo | No | Si | Si | Si | Si |
| RBAC | No | Si | Si | Si | Si |
| Replicacion | No | Si | No | No | No |
| Costo | Gratis | Gratis | Por uso | Por uso | Por uso |

> Los registries privados son el almacen seguro de las imagenes de contenedores. La eleccion entre self-hosted (Docker Registry, Harbor) y gestionados (ECR, Artifact Registry, ACR) depende del nivel de control necesario, la infraestructura existente y los requisitos de cumplimiento de la organizacion.

