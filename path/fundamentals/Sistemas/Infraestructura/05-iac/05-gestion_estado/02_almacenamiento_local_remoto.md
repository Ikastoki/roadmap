# Almacenamiento Local y Remoto del Estado

La decision sobre donde almacenar el archivo de estado tiene implicaciones directas en la capacidad de un equipo para trabajar de forma colaborativa, la disponibilidad de los datos de gestion y la resiliencia ante fallos. Por defecto, Terraform almacena el estado en un archivo local llamado `terraform.tfstate` dentro del directorio de ejecucion. Esta configuracion es funcional para pruebas individuales pero presenta riesgos criticos en entornos de produccion donde multiples personas o procesos automatizados interactuan con la misma infraestructura.

El almacenamiento local presenta limitaciones estructurales: el archivo reside en el disco duro del desarrollador, no esta versionado automaticamente, y no existe ningun mecanismo para prevenir que dos personas ejecuten planes simultaneos sobre el mismo conjunto de recursos. La perdida accidental del archivo, un fallo de disco o la destraccion del equipo de desarrollo resulta en la perdida completa del contexto de gestion.

El almacenamiento remoto resuelve estos problemas mediante la persistencia del estado en servicios de almacenamiento gestionados que ofrecen disponibilidad, durabilidad y capacidades de concurrencia. Los backends remotos mas utilizados incluyen Amazon S3, Google Cloud Storage, Azure Blob Storage y HashiCorp Cloud Platform, cada uno con caracteristicas especificas de configuracion y seguridad.

## Backend local por defecto

Cuando no se configura explicitamente un backend, Terraform utiliza el almacenamiento local. El archivo se genera automaticamente durante la primera ejecucion de `terraform apply` y se actualiza con cada operacion subsecuente.

```hcl
# No se requiere configuracion explicita para el backend local
# Terraform crea automaticamente terraform.tfstate en el directorio actual

# El archivo puede crecer significativamente con recursos complejos
# Se recomienda agregarlo al .gitignore
```

```bash
# El archivo de estado local se encuentra en:
ls -la terraform.tfstate

# Verificar su contenido con el CLI de Terraform
terraform show
```

> El archivo `terraform.tfstate` contiene informacion sensible como contraseñas, claves de acceso y tokens de autenticacion. Nunca debe ser versionado en repositorios publicos ni compartido sin los mecanismos de encriptacion apropiados.

## Backend S3 con DynamoDB

Amazon S3 es el backend remoto mas comun para equipos que trabajan en AWS. La combinacion de S3 para almacenamiento y DynamoDB para locking de estado ofrece una solucion robusta con encriptacion at-rest, versionado de objetos y control de acceso granular.

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state"
    key            = "produccion/infraestructura.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    
    # Opcional: encriptacion con KMS gestionado
    # kms_key_id = "alias/terraform-state"
  }
}
```

> La politica de acceso al bucket S3 debe seguir el principio de privilegio minimo. Solo los procesos de CI/CD y los desarrolladores autorizados deben tener permisos `s3:GetObject` y `s3:PutObject` sobre la ruta especifica del proyecto.

## Backend Google Cloud Storage

Google Cloud Storage ofrece capacidades similares a S3 con integracion nativa en el ecosistema GCP. El bucket debe tener habilitado el versionado y la encriptacion por defecto. El locking se gestiona mediante un bucket dedicado o un archivo de interbloqueo.

```hcl
terraform {
  backend "gcs" {
    bucket = "mi-empresa-terraform-state-prod"
    prefix = "infraestructura"
    
    # Encriptacion con KMS de Google Cloud
    # encryption_key = "projects/mi-proyecto/locations/global/keyRings/terraform/cryptoKeys/state"
  }
}
```

```bash
# Crear el bucket con versionado habilitado
gsutil versioning set on gs://mi-empresa-terraform-state-prod

# Configurar politica de retencion (90 dias recomendado)
gsutil lifecycle set lifecycle.json gs://mi-empresa-terraform-state-prod
```

## Backend Azure Blob Storage

Azure Blob Storage utiliza contenedores con un esquema de nombres jerarquico. El locking se gestiona a traves de blobs con lease bloqueado, una funcionalidad nativa del servicio de blobs de Azure.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "tfstateproduccion"
    container_name       = "tfstate"
    key                  = "infraestructura.tfstate"
    
    # Encriptacion con clave gestionada por Azure
    use_azuread_auth     = true
    subscription_id      = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    tenant_id            = "f6e5d4c3-b2a1-0987-fedc-ba0987654321"
  }
}
```

```bash
# Crear el grupo de recursos y la cuenta de almacenamiento
az group create --name rg-terraform-state --location westeurope
az storage account create \
  --name tfstateproduccion \
  --resource-group rg-terraform-state \
  --location westeurope \
  --sku Standard_LRS \
  --encryption-services blob

# Crear el contenedor
az storage container create \
  --name tfstate \
  --account-name tfstateproduccion
```

> Al migrar de un backend local a uno remoto, ejecutar `terraform init` activara un asistente de migracion que transferira el estado existente al nuevo almacenamiento. Es imperativo crear un respaldo del archivo local antes de iniciar este proceso.

## Comparativa de backends

| Caracteristica | S3 | GCS | Azure Blob |
|---|---|---|---|
| Encriptacion at-rest | SSE-S3, SSE-KMS | CMEK, CSEK | SSE, CMK |
| Versionado | Nativo | Nativo | Nativo |
| Locking | DynamoDB | File lock | Blob lease |
| Acceso IAM | IAM nativo | IAM nativo | Azure AD/RBAC |
| Multi-region | S3 Replication | Multi-region | GRS/RA-GRS |

