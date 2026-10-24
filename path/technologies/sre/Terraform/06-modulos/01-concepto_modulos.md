# Concepto de Modulos en Terraform

Un proyecto de Terraform que gestiona varios entornos con decenas de recursos pronto revela una realidad: bloques enteros de configuracion se repiten con ligeras variaciones. Un grupo de seguridad identico salvo por el rango CIDR, un conjunto de subnets que solo cambian en la zona de disponibilidad, una configuracion de balanceador que se replica en cada entorno. Escribir estos recursos una y otra vez no solo es tedioso, sino que introduce riesgo de desviaciones entre copias. Los modulos son el mecanismo que Terraform proporciona para encapsular, parametrizar y reutilizar configuraciones, elevando el codigo de infraestructura al mismo nivel de abstraccion que las funciones o clases en un lenguaje de programacion.

## Que es un modulo

Un modulo no es mas que un directorio que contiene archivos `.tf`. Cualquier directorio con codigo Terraform es, tecnicamente, un modulo. La distincion clave es intencional: un modulo bien disenado se crea para ser reutilizado, no para ejecutarse directamente como configuracion final. Un modulo agrupa recursos que tienen una relacion logica --una VPC con sus subnets, un cluster de Kubernetes, una base de datos replicada-- y los presenta como una unidad cohesiva que puede instanciarse multiples veces con distintos parametros.

La analogia con la programacion es precisa. Un modulo es comparable a una funcion: recibe argumentos de entrada (variables), ejecuta una logica interna (recursos) y devuelve valores de salida (outputs). Quien utiliza el modulo no necesita conocer los detalles internos de como se implementan los recursos, solo la interfaz que expone: que variables espera y que outputs produce.

```hcl
module "vpc_principal" {
  source = "./modules/vpc"

  nombre      = "produccion"
  cidr_bloque = "10.0.0.0/16"
  entornos    = ["dev", "staging", "prod"]
}
```

Cada bloque `module` crea una instancia independiente del modulo, con su propio conjunto de recursos. Esto permite reutilizar la misma logica de VPC para crear redes aisladas para distintos entornos o equipos sin duplicar codigo.

## Modulo root vs modulos hijos

Toda configuracion de Terraform se ejecuta desde un modulo root. El modulo root es el directorio desde el que se invocan los comandos `terraform plan` o `terraform apply`. Normalmente contiene la configuracion de alto nivel: el backend, los providers y las llamadas a modulos hijos que componen la infraestructura completa.

Los modulos hijos son aquellos que se invocan mediante bloques `module` desde el modulo root o desde otros modulos. No se ejecutan directamente; existen para ser llamados. La relacion entre root e hijos es estrictamente jerarquica: el root no puede ser llamado como hijo de otro modulo (aunque tecnicamente si se puede, es una practica desaconsejada que rompe la claridad del arbol de dependencias).

```hcl
# modulo root - main.tf
module "red" {
  source = "./modules/red"
  cidr   = "10.0.0.0/16"
}

module "base_datos" {
  source = "./modules/base_datos"
  vpc_id = module.red.vpc_id
}
```

El modulo root orquesta la composicion. Los modulos hijos implementan piezas especificas. Esta separacion permite que diferentes equipos sean duenos de distintos modulos, y que cada modulo evolucione de forma independiente siempre que mantenga su interfaz contractuada.

## Estructura de un modulo

Un modulo bien organizado sigue una estructura de archivos convencional que separa claramente las responsabilidades de cada componente. Aunque Terraform no exige nombres especificos de archivos (todo `.tf` en el directorio se evalua en conjunto), la comunidad ha adoptado una convencion que favorece la legibilidad y el mantenimiento.

```
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
└── README.md
```

**main.tf**: contiene los recursos principales del modulo. Es el nucleo de la logica: definiciones de recursos, data sources y cualquier bloque que describa la infraestructura que el modulo gestiona. Este archivo debe centrarse en el "que" y no en el "como se parametriza"; los valores configurables se delegan a variables.

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr_bloque

  tags = {
    Name = var.nombre
  }
}

resource "aws_subnet" "this" {
  count = length(var.subredes_cidr)

  vpc_id     = aws_vpc.this.id
  cidr_block = var.subredes_cidr[count.index]
  availability_zone = var.zonas[count.index]
}
```

**variables.tf**: declara las entradas que el modulo acepta. Cada variable debe incluir tipo, descripcion y, cuando tenga sentido, un valor por defecto. Las variables constituyen el contrato publico del modulo; quien lo usa debe conocer que parametros son obligatorios y cuales opcionales.

```hcl
# modules/vpc/variables.tf
variable "nombre" {
  description = "Nombre identificativo de la VPC"
  type        = string
}

variable "cidr_bloque" {
  description = "Bloque CIDR principal de la VPC"
  type        = string
}

variable "subredes_cidr" {
  description = "Lista de bloques CIDR para las subredes"
  type        = list(string)
  default     = []
}

variable "zonas" {
  description = "Lista de zonas de disponibilidad"
  type        = list(string)
  default     = null
}
```

**outputs.tf**: define los valores que el modulo expone al exterior. Los outputs permiten que otros modulos o el modulo root accedan a atributos de los recursos creados internamente, como IDs, ARNs o direcciones IP. Un modulo debe exponer solo la informacion necesaria para que sus consumidores puedan integrarse con el, sin filtrar detalles internos irrelevantes.

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID de la VPC creada"
  value       = aws_vpc.this.id
}

output "subredes_ids" {
  description = "IDs de las subredes creadas"
  value       = aws_subnet.this[*].id
}
```

**versions.tf**: establece las restricciones de versiones de Terraform y providers que el modulo requiere. Es especialmente importante cuando un modulo depende de features introducidas en versiones especificas, o cuando se quiere garantizar compatibilidad con un rango conocido de versiones.

```hcl
# modules/vpc/versions.tf
terraform {
  required_version = ">= 1.3.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0, < 6.0"
    }
  }
}
```

**README.md**: documento que explica el proposito del modulo, sus variables, outputs y un ejemplo de uso. Un README bien escrito diferencia un modulo util de uno inaccesible. Debe responder a tres preguntas: que hace este modulo, que parametros necesita y como se usa en la practica.

