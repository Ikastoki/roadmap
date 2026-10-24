# Proveedores y Registry

La potencia de Terraform no reside solo en su motor central, sino en la amplitud de su ecosistema de proveedores. Cada provider es una pieza de software independiente que sabe como comunicarse con la API de un servicio determinado, exponiendo sus recursos y fuentes de datos como bloques de HCL. El Terraform Registry actua como el punto de distribucion centralizado donde estos proveedores se publican, versionan y documentan.

## Providers

Un provider es un plugin que implementa el ciclo de vida CRUD (Create, Read, Update, Delete) para cada recurso de un servicio. Cuando Terraform encuentra un bloque `resource` de tipo `aws_instance`, delega al provider de AWS la responsabilidad de traducir esa declaracion en las llamadas API correspondientes.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

Cada provider requiere configuracion especifica: credenciales, regiones, endpoints personalizados o archivos de configuracion. Terraform busca estas credenciales en multiples fuentes siguiendo una jerarquia bien definida: variables explicitas en el bloque `provider`, variables de entorno, archivos de credenciales por defecto del proveedor (como `~/.aws/credentials`), y finalmente roles o identidades asignadas (como instance profiles en AWS o service accounts en GCP).

La directiva `required_providers` dentro del bloque `terraform` especifica que proveedores necesita el proyecto, su origen (source) y las restricciones de version. El source sigue el formato `namespace/type`: `hashicorp/aws` indica que el proveedor pertenece al namespace oficial de HashiCorp, mientras que `cloudflare/cloudflare` indica un proveedor mantenido por Cloudflare. Esta estructura de namespaces permite que cualquier organizacion publique sus propios proveedores sin conflictos de nombres.

### Versionado de providers

El versionado semantico de los providers es critico para la estabilidad de los proyectos. Terraform permite especificar restricciones de version con operadores:

- `= 5.0.0`: version exacta
- `~> 5.0`: cualquier version 5.x pero no 6.0
- `>= 2.0, < 3.0`: rango de versiones
- `>= 1.0`: cualquier version igual o superior a 1.0

> Fijar versiones de providers es una buena practica obligatoria en entornos de produccion. Una actualizacion automatica de un provider puede introducir cambios en el comportamiento de recursos existentes sin previo aviso.

### Providers oficiales, asociados y comunitarios

HashiCorp clasifica los providers en tres categorias:

**Oficiales**: mantenidos por HashiCorp (AWS, Azure, GCP, Kubernetes, GitHub, etc.). Son los mas utilizados y reciben actualizaciones frecuentes con soporte completo.

**Asociados**: mantenidos por el propio proveedor del servicio (Cloudflare, Datadog, MongoDB Atlas, Auth0, etc.). La empresa responsable del servicio garantiza la compatibilidad con su API.

**Comunitarios**: desarrollados y mantenidos por la comunidad de Terraform. No tienen garantias de soporte ni actualizaciones regulares, pero cubren una larga cola de servicios especializados.

## Terraform Registry

El Terraform Registry, accesible en `registry.terraform.io`, es el repositorio publico donde se alojan y distribuyen tanto providers como modulos. Funciona como un catalogo searchable donde cada entrada incluye documentacion completa, ejemplos de uso, historial de versiones y requisitos.

Para los providers, el Registry actua como la fuente de verdad: cuando `terraform init` se ejecuta, el core consulta el Registry para descargar las versiones especificadas de cada provider. La integridad de las descargas se verifica mediante checksums, y Terraform genera un archivo de bloqueo (`.terraform.lock.hcl`) que fija las versiones exactas descargadas para garantizar reproducibilidad.

```hcl
# .terraform.lock.hcl -- generado automaticamente
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.17.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:r4/5rP9Cw7wNRAm5JWxMhF9PAEI=",
  ]
}
```

Para los modulos, el Registry funciona como una biblioteca de componentes reutilizables. Cualquier repositorio de GitHub que siga la estructura de modulo de Terraform y cumpla con las convenciones de nomenclatura puede publicarse en el Registry. Los modulos mas populares incluyen `terraform-aws-modules/vpc/aws` para VPCs en AWS, `terraform-google-modules/kubernetes-engine/google` para GKE, o `Azure/terraform-azurerm-virtual-network` para Azure.

### Uso de modulos desde el Registry

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "mi-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Environment = "production"
  }
}
```

El formato de source para modulos del Registry es `namespace/module-name/provider-name`. El versionado sigue las mismas reglas que los providers, permitiendo fijar versiones especificas o rangos. Terraform descarga el modulo y lo almacena en el directorio `.terraform/modules/` durante la inicializacion.

### Fuentes alternativas de modulos

Aunque el Registry es la fuente mas comun, Terraform soporta otros origenes para modulos:

- **Local**: `source = "./modules/vpc"` para modulos en el mismo repositorio
- **Git**: `source = "git::https://github.com/org/repo.git?ref=v1.0.0"`
- **Mercurial**: `source = "hg::https://..."
- **HTTP**: `source = "https://example.com/module.tar.gz"`
- **S3/GCS**: `source = "s3::https://bucket/module.zip"`

Cada origen permite parametros adicionales como `ref` para ramas, tags o commits en Git, o `archive` para descargar de buckets de almacenamiento. Esta flexibilidad permite adaptar la estrategia de distribucion a las politicas de cada organizacion.

