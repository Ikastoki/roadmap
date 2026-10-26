# Arquitectura de una Herramienta IaC

Detras de la sintaxis declarativa que el usuario escribe existe una maquinaria compleja compuesta por cuatro componentes fundamentales: el motor o CLI, los providers o plugins, el estado y el registry. Estos componentes interactuan de forma coordinada para traducir la configuracion abstracta en llamadas reales a APIs de proveedores, manteniendo un registro persistente de que recursos existen y como se relacionan entre si. Comprender esta arquitectura permite diagnosticar problemas, optimizar despliegues y tomar decisiones informadas sobre que componente es responsable de cada fallo.

## Motor / CLI

El motor es el binario central que interpreta el codigo escrito por el usuario, construye un grafo de dependencias y orquesta las acciones necesarias para alcanzar el estado deseado. En Terraform, este motor se ejecuta mediante comandos como `terraform plan`, `terraform apply` o `terraform destroy`. El motor no contiene conocimiento sobre APIs de proveedores; delega esa responsabilidad completamente a los providers.

```bash
# Inicializacion del motor - descarga providers y modulos
terraform init

# Analisis del plan - calcula cambios sin aplicarlos
terraform plan -out=plan.bin

# Aplicacion del plan - ejecuta los cambios en el proveedor
terraform apply plan.bin

# Destruccion - elimina todos los recursos gestionados
terraform destroy
```

```bash
# Estructura basica de un proyecto IaC
my-project/
├── main.tf           # Definiciones de recursos principales
├── variables.tf      # Declaracion de variables de entrada
├── outputs.tf        # Valores exportados
├── providers.tf      # Configuracion de providers
├── terraform.tfstate # Estado local (no recomendado en produccion)
└── .terraform/       # Providers descargados por init
    └── providers/
        └── registry.terraform.io/
            └── hashicorp/
                └── aws/
                    └── 5.0.0/
                        └── linux_amd64/
                            └── terraform-provider-aws_v5.0.0
```

> El comando `terraform init` es el unico que modifica la estructura del proyecto local. Descarga providers, modulos y configura backends. Debe ejecutarse siempre antes de plan o apply, especialmente en CI/CD donde el directorio de trabajo empieza limpio.

## Providers / Plugins

Los providers son plugins que implementan la traduccion entre el lenguaje declarativo y las APIs de un servicio o proveedor. Cada provider expone un conjunto de recursos, data sources y funciones que el motor puede utilizar. Terraform mantiene un registry oficial donde los providers estan versionados y documentados.

```hcl
# Configuracion de providers en Terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "terraform"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

```hcl
# Providers anidados -AWS provider con perfiles distintos
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "logs_us" {
  provider = aws.us_east
  bucket   = "logs-us-east"
}

resource "aws_s3_bucket" "logs_eu" {
  provider = aws.eu_west
  bucket   = "logs-eu-west"
}
```

> Un provider no es lo mismo que un recurso. El provider es el conector con la API; el recurso es la abstraction de un objeto concreto que ese provider puede crear. El provider `hashicorp/aws` puede gestionar cientos de recursos distintos: `aws_instance`, `aws_s3_bucket`, `aws_lambda_function`, etc.

## Estado

El estado es la base de datos interna que la herramienta utiliza para mapear la configuracion declarada con los recursos fisicos existentes. Cada vez que se ejecuta un plan, el motor compara el estado actual contra la definicion del codigo para determinar que recursos deben crearse, modificarse o eliminarse. Sin estado, la herramienta no podria saber que ya creo un recurso en ejecuciones anteriores.

```hcl
# Configuracion de backend remoto (S3 + DynamoDB)
terraform {
  backend "s3" {
    bucket         = "mi-terraform-state"
    key            = "produccion/infraestructura.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

```json
// Estructura simplificada del archivo de estado
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 12,
  "lineage": "abc123-def456",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t3.micro",
            "public_ip": "54.123.45.67",
            "private_ip": "10.0.1.15"
          }
        }
      ]
    }
  ]
}
```

> El estado nunca debe almacenarse en repositorios Git. Contiene informacion sensible (IPs, IDs, contraseñas en algunos casos) y su formato puede cambiar entre versiones. Usa backends remotos con bloqueo distribuido para entornos de equipo (S3+DynamoDB, Consul, Terraform Cloud, etc.).

## Registry

El registry es un repositorio centralizado de modulos y providers reutilizables. Terraform Registry (registry.terraform.io) es el oficial, pero existen registros privados como Terraform Enterprise, Artifactory o registros genericos de OCI. Los modulos encapsulan conjuntos de recursos logicamente agrupados y exponen variables para parametrizar su comportamiento.

```hcl
# Uso de un modulo desde el registry publico
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "mi-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}
```

```bash
# Comandos para gestionar modulos y providers
terraform init -upgrade          # Actualizar providers y modulos
terraform providers mirror ./mirrors  # Crear mirror local de providers
terraform state list             # Listar recursos en el estado
terraform state show aws_instance.web  # Ver detalles de un recurso
```

## Flujo de Ejecucion Completo

El diagrama siguiente muestra como interactuan los cuatro componentes durante un ciclo tipico de ejecucion. El usuario escribe codigo HCL, el motor lo interpreta, consulta el estado actual, resuelve las dependencias entre recursos y delega las llamadas API a los providers correspondientes.

```
┌─────────────────────────────────────────────────────────────────┐
│                        USUARIO                                  │
│   main.tf + variables.tf + terraform.tfvars                     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    MOTOR / CLI                                   │
│   1. Parsea el codigo HCL                                       │
│   2. Construye grafo de dependencias                            │
│   3. Lee estado actual                                           │
│   4. Calcula diff (plan)                                        │
│   5. Ejecuta acciones (apply/destroy)                           │
└───────┬──────────────────┬──────────────────────┬───────────────┘
        │                  │                      │
        ▼                  ▼                      ▼
┌──────────────┐  ┌────────────────┐  ┌────────────────────┐
│   ESTADO     │  │   PROVIDERS    │  │    REGISTRY         │
│              │  │                │  │                    │
│ S3/Consul/   │  │ hashicorp/aws  │  │ terraform-aws-     │
│ TF Cloud     │  │ hashicorp/azur │  │ modules/vpc/aws    │
│              │  │ hashicorp/gcp  │  │                    │
│ Mapea:       │  │                │  │ Modulos:           │
│ config →     │  │ Traduce:       │  │ vpc, rds, eks...   │
│ recurso real │  │ HCL → API call │  │                    │
└──────────────┘  └────────────────┘  └────────────────────┘
        │                  │                      │
        │                  ▼                      │
        │     ┌─────────────────────┐            │
        └────▶│   PROVEEDOR CLOUD   │◀───────────┘
              │                     │
              │ AWS / Azure / GCP / │
              │ Kubernetes / etc.   │
              │                     │
              │ Recursos fisicos:   │
              │ VMs, VPCs, DBs...   │
              └─────────────────────┘
```

> Este modelo de arquitectura no es exclusivo de Terraform. OpenTofu comparte la misma estructura, Pulumi reemplaza el lenguaje HCL por lenguajes generales pero mantiene providers y estado, y Crossplane usa Kubernetes como motor y etcd como estado. La separacion de responsabilidades es un patron constante en todo el ecosistema IaC.

