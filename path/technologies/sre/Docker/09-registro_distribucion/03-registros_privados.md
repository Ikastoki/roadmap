# Registros privados de imagenes

Docker Hub resuelve la necesidad basica de almacenamiento y distribucion de imagenes, pero las organizaciones con requisitos de seguridad, cumplimiento normativo o simplemente con un volumen elevado de imagenes privadas necesitan soluciones que ofrezcan control sobre el almacenamiento, la autenticacion y la auditoria. Los registros privados cubren ese espectro: desde el registro OSS minimo que cualquiera puede desplegar en un servidor hasta servicios gestionados en la nube con integracion nativa en el ecosistema del proveedor.

## Docker Registry: el registro OSS

Docker Registry es la implementacion de referencia del protocolo OCI Distribution. Es un servicio estateless escrito en Go que puede desplegarse como un contenedor y configurarse con distintos backends de almacenamiento: sistema de archivos local, S3, Azure Blob Storage, Google Cloud Storage o Swift.

```bash
# Ejecutar un registry local en el puerto 5000
docker run -d -p 5000:5000 --name registry registry:2

# Etiquetar y subir una imagen al registry local
docker tag mi-app:latest localhost:5000/mi-app:1.0.0
docker push localhost:5000/mi-app:1.0.0
```

Por defecto, Docker Registry no incluye autenticacion ni cifrado. En produccion debe ejecutarse detras de un proxy reverso (Nginx, Traefik) que termine TLS y proporcione autenticacion basica o mediante token. Tampoco tiene interfaz web ni funciones de escaneo; es un componente minimalista que resuelve exclusivamente el almacenamiento y la distribucion.

```yaml
# config.yml para registry con autenticacion basica y almacenamiento S3
version: 0.1
storage:
  s3:
    accesskey: AKIA...
    secretkey: ...
    bucket: mi-registry
    region: eu-west-1
    rootdirectory: /docker
http:
  addr: :5000
  tls:
    certificate: /certs/domain.crt
    key: /certs/domain.key
auth:
  htpasswd:
    realm: basic-realm
    path: /auth/htpasswd
```

Docker Registry es adecuado para equipos pequenos o entornos de desarrollo que necesitan un registro interno sin complicaciones. Para entornos empresariales, sus carencias en seguridad y gestion lo convierten en una opcion menos recomendable.

## Harbor: el registro empresarial open source

Harbor es un proyecto de Cloud Native Computing Foundation (CNCF) que extiende Docker Registry con funcionalidades empresariales: interfaz web, control de acceso basado en roles (RBAC), replicacion entre registros, escaneo de vulnerabilidades integrado y firmado de imagenes con Notary.

La arquitectura de Harbor anade varios componentes al nucleo del registry. La interfaz web permite gestionar proyectos, usuarios y politicas sin necesidad de lineas de comando. Cada proyecto puede configurarse como publico o privado, y los usuarios se asignan con roles que van desde `guest` (solo lectura) hasta `projectAdmin` (gestion completa del proyecto).

```bash
# Desplegar Harbor con docker-compose
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar xzf harbor-online-installer-v2.10.0.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml
# Configurar hostname, certificados y contrasenas en harbor.yml
./install.sh
```

La replicacion es una de las funcionalidades mas potentes de Harbor. Permite configurarpoliticas que copian imagenes entre instancias de Harbor o desde/hacia registros externos (Docker Hub, ECR, GCR, ACR). Esto es util para distribuir imagenes entre regiones, mantener un registro de respaldo o sincronizar imagenes desde un registro publico a uno privado para evitar dependencias externas.

```yaml
# Politica de replicacion en Harbor
name: replicar-a-dr
source:
  registry: harbor-principal.miempresa.com
  filter:
    type: repository
    pattern: "produccion/**"
destination:
  registry: harbor-dr.miempresa.com
replication:
  mode: event-driven
  override: true
```

Harbor tambien incluye un escaner de vulnerabilidades integrado que puede utilizar Trivy, Clair o Aqua Security como motor. Cuando una imagen se sube al registro, Harbor la escanea automaticamente y etiqueta el repositorio con el nivel de gravedad de las vulnerabilidades encontradas. Las politicas de inmuta bilidad (immutable tags) evitan que una etiqueta existente se sobrescriba, lo que garantiza la trazabilidad en produccion.

## AWS ECR: registro gestionado en AWS

Amazon Elastic Container Registry (ECR) es el registro de contenedores nativo de AWS. Se integra con el ecosistema IAM para la autenticacion y autorizacion, y almacena las imagenes en buckets de S3 gestionados por AWS. ECR no cobra por el almacenamiento (solo por el transferencia de datos y el tiempo de almacenamiento), y las imagenes se replican automaticamente en las tres zonas de disponibilidad de cada region.

La autenticacion en ECR se realiza mediante tokens temporales generados con AWS CLI:

```bash
# Obtener token de autenticacion para ECR
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.eu-west-1.amazonaws.com

# Crear un repositorio
aws ecr create-repository --repository-name mi-app --region eu-west-1

# Etiquetar y subir
docker tag mi-app:latest <account-id>.dkr.ecr.eu-west-1.amazonaws.com/mi-app:1.0.0
docker push <account-id>.dkr.ecr.eu-west-1.amazonaws.com/mi-app:1.0.0
```

ECR ofrece politicas de ciclo de vida que automaticamente eliminan imagenes antiguas basandose en reglas de cantidad o tiempo. Esto evita la acumulacion de imagenes obsoletas y controla los costos de almacenamiento.

```json
// Politica de ciclo de vida: mantener las ultimas 5 imagenes
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Mantener solo 5 imagenes por repositorio",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

La integracion con AWS Identity and Access Management (IAM) permite controlar quien puede hacer push y pull a nivel de repositorio mediante politicas de recursos. Por ejemplo, es posible otorgar acceso de solo lectura a un rol de CI/CD y acceso de escritura a un rol de construccion, todo sin gestionar contrasenas adicionales.

## Google Container Registry y Artifact Registry

Google Cloud ofrece dos servicios de registro: Container Registry (gcr.io), el servicio clasico, y Artifact Registry, su sucesor que unifica el almacenamiento de contenedores, paquetes Maven, npm y Python.

Artifact Registry es el servicio recomendado por Google. Proporciona integracion con IAM, escaneo de vulnerabilidades con Cloud Build y almacenamiento regional o multirregional. La autenticacion se realiza mediante gcloud CLI:

```bash
# Configurar autenticacion para Artifact Registry
gcloud auth configure-docker europe-west1-docker.pkg.dev

# Etiquetar y subir
docker tag mi-app:latest \
  europe-west1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:1.0.0
docker push europe-west1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:1.0.0
```

Artifact Registry permite configurar politicas de limpieza automatica y replicacion entre regiones. Al estar integrado con Cloud Build, las imagenes pueden construirse y almacenarse sin necesidad de credenciales externas.

## Azure Container Registry

Azure Container Registry (ACR) es el registro gestionado de Microsoft Azure. Ofrece tres SKUs: Basic, Standard y Premium, que se diferencian en el almacenamiento, el rendimiento y las funciones adicionales como la replicacion geografica y el firewall de red.

```bash
# Iniciar sesion en ACR
az acr login --name miacr

# Etiquetar y subir
docker tag mi-app:latest miacr.azurecr.io/mi-app:1.0.0
docker push miacr.azurecr.io/mi-app:1.0.0
```

ACR se integra con Azure Active Directory para la autenticacion, permitiendo asignar permisos a nivel de repositorio mediante roles RBAC. La replicacion geografica del tier Premium replica automaticamente las imagenes en multiples regiones para reducir la latencia de descarga en despliegues globales. ACR tambien incluye escaneo de vulnerabilidades basado en Trivy en los planes Basic y superiores.

## Criterios para elegir un registro privado

La eleccion del registro depende del contexto de la organizacion. Cuando la infraestructura ya reside en un proveedor de nube, el registro nativo (ECR, Artifact Registry, ACR) ofrece la integracion mas natural con IAM y el ciclo de vida del proveedor. Para entornos multicloud u on-premise, Harbor proporciona una plataforma homogenea que funciona independientemente del proveedor de infraestructura.

El registro OSS de Docker es adecuado para laboratorios o equipos muy pequenos, pero su falta de autenticacion nativa, auditoria y escaneo lo descarta para entornos con requisitos de compliance. En esos casos, la inversion en Harbor o en un registro gestionado se justifica por la trazabilidad, el control de acceso y la automatizacion de la limpieza de imagenes.

