# Escritura de configuracion

La fase de escritura es donde el ingeniero traduce la arquitectura de infraestructura deseada en codigo declarativo. Cada archivo define que recursos existen, que propiedades tienen y como se relacionan entre si. A diferencia de los scripts imperativos que describen pasos para llegar a un resultado, la configuracion de IaC describe el estado final deseado; el motor se encarga de determinar que operaciones son necesarias para alcanzarlo. La calidad de la configuracion escrita determina directamente la mantenibilidad, reusabilidad y fiabilidad de toda la infraestructura gestionada.

## Definicion de proveedores y versiones

Cada proyecto debe comenzar especificando los proveedores que utilizara y las versiones exactas de cada uno. El bloque `required_providers` establish las restricciones de version, y el bloque `required_version` determina la version minima del propio motor de IaC. Esta pratctica garantiza reproducibilidad: cualquier miembro del equipo que ejecute la configuracion usara exactamente los mismos binarios.

```hcl
# providers.tf
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.31"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

El operador `~>` (pessimistic constraint) permite actualizaciones de parche y menores pero bloquea cambios de version mayor. Esto equilibra estabilidad con acceso a mejoras no destructivas. Para entornos criticos, se pueden fijar versiones exactas con `= 5.31.0`, aunque esto requiere actualizaciones manuales.

## Declaracion de variables

Las variables parametrizan la configuracion, permitiendo reutilizar el mismo codigo para multiples entornos o contextos. Cada variable puede incluir un tipo, valor por defecto, descripcion y validaciones que se ejecutan antes de cualquier llamada a la API.

```hcl
# variables.tf
variable "aws_region" {
  description = "Region de AWS para desplegar los recursos"
  type        = string
  default     = "eu-west-1"

  validation {
    condition     = can(regex("^eu-|^us-|^ap-", var.aws_region))
    error_message = "La region debe comenzar con eu-, us- o ap-."
  }
}

variable "environment" {
  description = "Nombre del entorno (dev, staging, production)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "El entorno debe ser dev, staging o production."
  }
}

variable "instance_type" {
  description = "Tipo de instancia EC2"
  type        = string
  default     = "t3.micro"
}

variable "vpc_cidr" {
  description = "Bloque CIDR de la VPC principal"
  type        = string
  default     = "10.0.0.0/16"
}
```

> Las validaciones de variables son evaluadas durante `plan` y `apply`. Si alguna validacion falla, el motor aborta antes de realizar cualquier cambio en la infraestructura real. Esto actua como una primera linea de defensa contra configuraciones invalidas.

## Declaracion de recursos

Los bloques de recurso representan los componentes tangibles de la infraestructura: maquinas virtuales, bases de datos, redes, balances de carga. Cada recurso incluye un tipo (que determina que API se invoca) y un identificador local (que el motor usa para referenciar el recurso en otros bloques).

```hcl
# main.tf
resource "aws_vpc" "principal" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "vpc-${var.environment}"
  }
}

resource "aws_subnet" "publica" {
  vpc_id                  = aws_vpc.principal.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, 1)
  map_public_ip_on_launch = true
  availability_zone       = "${var.aws_region}a"

  tags = {
    Name = "subnet-publica-${var.environment}"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  subnet_id     = aws_subnet.publica.id

  tags = {
    Name = "web-${var.environment}"
  }
}
```

La expresion `aws_subnet.publica.id` demuestra la resolucion implicita de dependencias: el motor detecta que la subnet depende de la VPC, y que la instancia depende de la subnet, y ejecuta las operaciones en el orden correcto sin que el usuario lo especifique explicitamente.

## Declaracion de outputs

Los outputs exponen valores derivados de los recursos creados. Son la interfaz de salida del proyecto, permitiendo que otros modulos, scripts o herramientas consumean informacion sobre la infraestructura desplegada.

```hcl
# outputs.tf
output "vpc_id" {
  description = "ID de la VPC principal"
  value       = aws_vpc.principal.id
}

output "subnet_publica_id" {
  description = "ID de la subnet publica"
  value       = aws_subnet.publica.id
}

output "instance_public_ip" {
  description = "IP publica de la instancia web"
  value       = aws_instance.web.public_ip
}

output "resumen_entorno" {
  description = "Resumen del entorno desplegado"
  value = {
    vpc_id    = aws_vpc.principal.id
    subnet_id = aws_subnet.publica.id
    ip        = aws_instance.web.public_ip
    region    = var.aws_region
  }
}
```

> Los outputs se muestran al final de cada ejecucion de `plan` y `apply`, y se almacenan en el estado. Los valores marcados como `sensitive = true` no se mostraran en texto plano en la terminal.

## Estructura de archivos

Un proyecto bien organizado separa la configuracion en archivos por responsabilidad. Esta separacion no tiene efecto funcional — el motor procesa todos los `.tf` del directorio como un unico conjunto — pero mejora drasticamente la navegacion y el mantenimiento.

| Archivo          | Contenido                                                     |
| ---------------- | ------------------------------------------------------------- |
| `providers.tf`   | Bloque `terraform`, `required_providers`, bloques `provider`  |
| `variables.tf`   | Todas las declaraciones de `variable`                         |
| `main.tf`        | Recursos principales del proyecto                             |
| `outputs.tf`     | Todas las declaraciones de `output`                           |
| `backend.tf`     | Configuracion del backend de estado                           |
| `data.tf`        | Fuentes de datos (`data` blocks)                              |
| `locals.tf`      | Bloques `locals` con valores computados                       |

```
proyecto-web/
├── providers.tf
├── variables.tf
├── main.tf
├── outputs.tf
├── backend.tf
├── data.tf
├── locals.tf
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ecs/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Cuando el proyecto crece, los modulos permiten encapsular componentes reutilizables. Cada modulo tiene su propio conjunto de archivos `main.tf`, `variables.tf` y `outputs.tf`, y se invoca desde el directorio raiz mediante bloques `module` que pasan variables como argumentos y recogen outputs como valores de retorno.

