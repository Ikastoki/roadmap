# Conceptos clave

Terraform introduce un vocabulario propio que es necesario dominar para leer y escribir configuraciones con fluidez. Cada concepto representa una pieza del modelo mental que la herramienta utiliza para traducir codigo en infraestructura real. Entenderlos por separado y en conjunto es lo que diferencia a un usuario ocasional de alguien que disena arquitecturas completas con Terraform.

## Provider

Un provider es un plugin que expone los recursos y fuentes de datos de un servicio concreto. Actua como el adaptador entre Terraform y la API del proveedor: autentica las peticiones, traduce los bloques de HCL a llamadas REST o gRPC, y gestiona el ciclo de vida de cada recurso. Sin un provider declarado, Terraform no sabe como comunicarse con ninguna plataforma.

```hcl
provider "aws" {
  region = "us-east-1"
}
```

Cada provider tiene su propia documentacion, sus propios tipos de recursos y sus propios requisitos de autenticacion. Terraform soporta oficialmente proveedores para AWS (`aws`), Azure (`azurerm`), GCP (`google`), Kubernetes (`kubernetes`), GitHub (`github`), entre cientos mas. Los providers se declaran en el codigo y se descargan automaticamente durante `terraform init`.

## Resource

Un resource es la unidad fundamental de infraestructura en Terraform. Representa un componente real: una maquina virtual, un bucket de S3, una base de datos, un registro DNS, una politica de IAM. Cada resource pertenece a un provider y se define con un tipo y un nombre local.

```hcl
resource "aws_s3_bucket" "datos" {
  bucket = "mi-bucket-datos-2024"
  acl    = "private"
}
```

Terraform gestiona el ciclo de vida completo del resource: lo crea si no existe, lo modifica si la configuracion cambio, y lo destruye si se elimina del codigo. Los resources pueden depender unos de otros, y Terraform construye automaticamente un grafico de dependencias para determinar el orden optimo de operaciones.

## Module

Un modulo es un contenedor que agrupa multiples recursos relacionados y los presenta como una unidad reutilizable. Es el mecanismo de encapsulacion y abstraccion de Terraform. Cualquier directorio con archivos `.tf` es un modulo, pero el termino suele referirse a modulos diseñados para ser reutilizados.

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "mi-vpc"
  cidr = "10.0.0.0/16"
}
```

Los modulos aceptan variables de entrada y producen outputs, siguiendo el mismo patron que las funciones en programacion. Permiten crear bibliotecas de infraestructura que se comparten entre proyectos, equipos o incluso publicamente a traves del Terraform Registry.

## State

El estado es el archivo que Terraform utiliza para mantener el mapeo entre los recursos declarados en codigo y los recursos reales en los proveedores. Sin estado, Terraform no podria determinar que cambios aplicar. El state se almacena por defecto en un archivo local `terraform.tfstate`, pero en entornos profesionales se configura un backend remoto (S3, GCS, Azure Blob, Terraform Cloud) para permitir la colaboracion en equipo.

El estado contiene informacion sensible en muchos casos (IPs, IDs de recursos, y a veces valores de secretos), por lo que su proteccion es critica. Terraform permite cifrarlo en reposo cuando se usa un backend remoto con soporte de cifrado.

> El archivo de estado no debe editarse manualmente. Cualquier modificacion directa puede provocar desincronizacion entre el estado real y el registrado, llevando a comportamientos impredecibles en futuras ejecuciones.

## Plan

Un plan es una previsualizacion de los cambios que Terraform ejecutara para llevar la infraestructura del estado actual al estado declarado en la configuracion. Se genera con `terraform plan` y muestra, recurso por recurso, que se creara, modificara o destruira.

```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t2.micro"
      ...
    }
```

El plan es el mecanismo de seguridad fundamental de Terraform: permite revisar los cambios antes de aplicarlos, evitar sorpresas y establecer un punto de control en pipelines de CI/CD. Puede guardarse en un archivo (`terraform plan -out=plan.tfplan`) para aplicarlo posteriormente de forma identica.

## Workspace

Un workspace es una instancia aislada del estado de Terraform dentro del mismo directorio de trabajo. Permite gestionar multiples entornos (desarrollo, staging, produccion) con el mismo codigo fuente, manteniendo estados separados.

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select dev
```

Cada workspace tiene su propio archivo de estado, lo que permite desplegar la misma configuracion en distintos contextos sin mezclar recursos. La variable `terraform.workspace` esta disponible en las configuraciones para adaptar comportamientos segun el entorno.

## Backend

El backend determina donde se almacena el estado de Terraform y como se ejecutan las operaciones. El backend por defecto es `local`, que guarda el estado en el sistema de archivos local. Para trabajo en equipo es necesario un backend remoto que proporcione almacenamiento compartido y bloqueo de estado.

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```

Los backends remotos mas comunes son S3 (AWS), GCS (GCP), AzureRM Blob Storage y Terraform Cloud. Ademas de almacenar el estado de forma centralizada, ofrecen `state locking` para evitar modificaciones simultaneas que corrompan el estado.

## Variable

Las variables son los parametros de entrada de las configuraciones de Terraform. Permiten parametrizar codigo para hacerlo reutilizable en distintos contextos sin modificar los archivos principales.

```hcl
variable "instance_type" {
  description = "Tipo de instancia EC2"
  type        = string
  default     = "t3.micro"
}

variable "environment" {
  description = "Entorno de despliegue"
  type        = string
}
```

Las variables pueden asignarse de multiples formas: por defecto en su declaracion, mediante archivos `.tfvars`, a traves de variables de entorno (`TF_VAR_nombre`), o directamente en la linea de comandos con `-var`. Terraform evalua estas fuentes en un orden de precedencia bien definido, donde los valores explicitos en CLI tienen maxima prioridad.

## Output

Los outputs son los valores de retorno de una configuracion o modulo de Terraform. Exponen informacion util que otros modulos o sistemas externos pueden consumir: direcciones IP, nombres DNS, IDs de recursos, etc.

```hcl
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "IP publica de la instancia web"
  sensitive   = false
}
```

Marcar un output como `sensitive = true` oculta su valor en los logs de Terraform, aunque sigue siendo accesible mediante `terraform output`. Los outputs son el mecanismo principal para pasar informacion entre modulos o integrar Terraform con scripts y herramientas externas.

