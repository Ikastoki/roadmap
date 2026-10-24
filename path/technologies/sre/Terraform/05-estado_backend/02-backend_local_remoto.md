# Backend local y remoto

El backend determina donde se almacena el archivo de estado y como se accede a el. Terraform soporta multiples backends, cada uno con diferentes caracteristicas de persistencia, concurrencia y seguridad. La eleccion del backend es una de las decisiones arquitectonicas mas importantes en cualquier proyecto que use Terraform, especialmente cuando participa mas de una persona.

## Backend local

Por defecto, Terraform almacena el estado en un archivo llamado `terraform.tfstate` en el directorio de trabajo actual. Este comportamiento es suficiente para entornos de aprendizaje, pruebas locales o proyectos unipersonales donde el bloqueo y la comparticion no son necesarios.

```hcl
terraform {
  # Sin bloque backend -> uso del backend local por defecto
}
```

Caracteristicas del backend local:

- El archivo `terraform.tfstate` se genera en el directorio raiz del proyecto.
- No existe mecanismo de bloqueo: dos ejecuciones simultaneas desde la misma maquina pueden corromper el archivo.
- No hay comparticion integrada: compartir el estado requiere copiar manualmente el archivo (SCP, correo, repositorio Git), lo que es propenso a errores humanos.
- Es adecuado para laboratorios locales o pipelines CI/CD donde cada ejecucion es secuencial y efimera.

> Incluir `terraform.tfstate` en un repositorio Git es una practica riesgosa. El estado puede contener informacion sensible (contrasenas, claves privadas, ARNs de recursos) y su gestion de versiones no coincide con la del codigo, generando conflictos de merge dificiles de resolver.

## Backend remoto

Un backend remoto almacena el estado en un servicio externo, generalmente un bucket de objetos cloud, con ventajas significativas respecto al almacenamiento local. La configuracion se define dentro del bloque `terraform` del archivo principal:

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-org-terraform-state"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Amazon S3 (AWS)

El backend S3 almacena el estado en un bucket de Amazon S3. Es el backend remoto mas utilizado en el ecosistema AWS por su simplicidad, bajo costo y durabilidad.

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-123456789012"
    key            = "infra/compute/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    kms_key_id     = "alias/terraform-bucket-key"
    dynamodb_table = "terraform-locks"
  }
}
```

Ventajas especificas:
- Versionado de objetos S3 para historial de cambios.
- Cifrado en reposo con SSE-S3 o KMS.
- Politicas IAM para control de acceso.
- Integracion con DynamoDB para state locking.

### Google Cloud Storage (GCS)

El backend GCS ofrece caracteristicas similares mediante buckets de Google Cloud Storage, con la ventaja adicional de que el bloqueo de estado esta integrado en GCS sin necesidad de un servicio separado.

```hcl
terraform {
  backend "gcs" {
    bucket = "mi-org-terraform-state"
    prefix = "terraform/state"
  }
}
```

GCS soporta bloqueo nativo mediante objetos de bloqueo generacionales. No requiere una tabla DynamoDB ni otro servicio auxiliar.

### Azure Blob Storage

El backend de Azure utiliza Azure Blob Storage, generalmente desde un contenedor dentro de una storage account.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "RG-Infra"
    storage_account_name = "tfstate0123456789"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

El bloqueo se realiza mediante leases de Azure Blob Storage, proporcionando exclusion mutua a nivel de blob.

### Terraform Cloud

Terraform Cloud es el servicio SaaS de HashiCorp que ofrece backend gestionado, ejecucion remota de planes y applys, y colaboracion en equipo.

```hcl
terraform {
  cloud {
    organization = "mi-org"
    workspaces {
      name = "infra-produccion"
    }
  }
}
```

Terraform Cloud anade capacidades que los backends de object storage no ofrecen:
- Ejecucion remota: el plan y apply se ejecutan en los servidores de HashiCorp.
- Historial de ejecuciones con aprobaciones manuales.
- Variables sensibles gestionadas centralizadamente.
- Politicas de compliance con Sentinel.
- API REST para integracion con CI/CD.

## State locking

El state locking es el mecanismo que impide que dos ejecuciones simultaneas modifiquen el mismo archivo de estado. Sin locking, dos desarrolladores ejecutando `apply` al mismo tiempo provocarian que el ultimo en escribir sobrescribiera los cambios del primero, resultando en un estado inconsistente que no refleja la totalidad de las modificaciones.

Cada backend implementa el bloqueo de forma diferente:

- **S3 + DynamoDB**: DynamoDB almacena un registro de bloqueo con clave primaria igual a la clave del estado en S3. Cuando Terraform inicia una operacion que modifica el estado, crea una entrada en DynamoDB. Si otro proceso intenta lo mismo, recibe un error de `ConditionalCheckFailedException`.
- **GCS**: Utiliza bloqueo generacional de objetos. Terraform escribe un archivo de bloqueo temporal; si el archivo ya existe, la operacion falla.
- **Azure Blob**: Usa leases de blobs. Un lease adquirido impide que otros clientes modifiquen el blob hasta que se libere.

```hcl
# Configuracion tipica con locking via DynamoDB
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```

Cuando ocurre un conflicto, Terraform muestra un mensaje similar a:

```
Error: Error acquiring the state lock

Error message: the state was locked by another process that is currently
running. Lock ID: "abc123"
```

El usuario debe esperar a que la otra operacion termine o, si esta convencido de que el bloqueo es huerfano (por ejemplo, un proceso abortado), forzar la liberacion con `terraform force-unlock <LOCK_ID>`.

## Ventajas del backend remoto

| Aspecto              | Local                          | Remoto                                   |
|----------------------|--------------------------------|------------------------------------------|
| Colaboracion         | Manual, propenso a errores     | Automatica, todos ven el mismo estado    |
| Bloqueo              | No                             | Si (DynamoDB, GCS, Azure Lease)          |
| Historial            | No nativo                      | Versionado de objetos S3/GCS             |
| Seguridad            | Depende del sistema de archivos| Cifrado en reposo, IAM, politicas        |
| Disponibilidad       | Local                          | Alta disponibilidad del proveedor cloud  |
| Auditoria            | No                             | Logs de acceso, versiones anteriores     |

La migracion de local a remoto se realiza con `terraform init -migrate-state`, que copia el estado local al nuevo backend configurado y actualiza el archivo `.terraform/terraform.tfstate` con la nueva ubicacion.

