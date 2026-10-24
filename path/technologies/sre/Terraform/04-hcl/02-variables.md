# Variables

Las variables son el mecanismo de parametrizacion de Terraform. Sin variables, cada configuracion estaria ligada a valores concretos, lo que imposibilitaria la reutilizacion del mismo codigo en distintos entornos (desarrollo, pruebas, produccion) o contextos. Las variables permiten separar la logica de la configuracion de los valores especificos, siguiendo el principio de que el codigo debe ser generico y los datos deben ser inyectados desde fuera.

Terraform distingue entre variables de entrada (input variables), que son parametros que se pasan a la configuracion, y variables locales (local values), que son valores calculados internamente. Este capitulo se centra en las variables de entrada.

## Definicion de variables

Las variables se definen mediante el bloque `variable`. Aunque pueden declararse en cualquier archivo `.tf`, la convencion establece que se agrupen en un archivo `variables.tf` para facilitar su localizacion.

```hcl
variable "nombre_variable" {
  description = "Descripcion clara del proposito"
  type        = <tipo>
  default     = <valor_opcional>
  sensitive   = <true|false>
  validation  = <bloque_de_validacion>
}
```

Todos los parametros excepto `type` son opcionales. Si no se especifica `type`, Terraform asume `any`, lo que significa que acepta cualquier tipo de valor. Si no se especifica `default`, la variable es obligatoria y Terraform solicitara un valor antes de ejecutar cualquier operacion.

```hcl
variable "region_aws" {
  description = "Region de AWS donde se desplegaran los recursos"
  type        = string
  default     = "us-east-1"
}

variable "instancias" {
  description = "Numero de instancias a crear"
  type        = number
}

variable "tags" {
  description = "Etiquetas para los recursos"
  type        = map(string)
  default = {
    Proyecto = "Plataforma"
    Entorno  = "Produccion"
  }
}

variable "config_red" {
  description = "Configuracion de red"
  type = object({
    vpc_cidr        = string
    subnet_cidrs    = list(string)
    enable_nat      = bool
  })
}
```

> Si una variable es obligatoria (no tiene `default`), Terraform preguntara interactivamente por su valor durante `terraform plan` o `terraform apply` si no se ha proporcionado por otro medio. En entornos automatizados (CI/CD), esto provoca un fallo, por lo que todas las variables deben tener un valor asignado antes de la ejecucion.

### Tipos disponibles

El sistema de tipos de HCL para variables incluye los tipos primitivos (`string`, `number`, `bool`), los tipos de coleccion (`list(<tipo>)`, `map(<tipo>)`, `set(<tipo>)`) y los tipos estructurales (`object({ ... })`, `tuple([...])`). Tambien existe el tipo `any` para cuando no se quiere restringir el tipo.

```hcl
variable "cualquier_cosa" {
  type = any
}
```

El uso de `any` desactiva la validacion de tipos en tiempo de plan, lo que puede provocar errores inesperados en tiempo de apply. Se recomienda usarlo solo cuando sea estrictamente necesario.

## Asignacion de valores

Terraform proporciona multiples mecanismos para asignar valores a las variables, con una precedencia definida que determina que valor prevalece cuando coinciden varias fuentes. El orden de precedencia, de menor a mayor prioridad, es:

1. Valor por defecto (`default` en el bloque `variable`)
2. Variable de entorno `TF_VAR_<nombre>`
3. Archivo `terraform.tfvars` (auto-cargado)
4. Archivo `terraform.tfvars.json` (auto-cargado)
5. Archivos `*.auto.tfvars` o `*.auto.tfvars.json` (orden alfabetico)
6. Archivo `-var-file` explicito en CLI
7. Parametro `-var` en CLI

### Archivos .tfvars

Los archivos con extension `.tfvars` contienen asignaciones en sintaxis HCL. Son el metodo mas utilizado para separar configuraciones por entorno.

```hcl
# terraform.tfvars
region_aws = "eu-west-1"
instancias = 3
tags = {
  Proyecto = "Plataforma"
  Entorno  = "Desarrollo"
}
```

Terraform carga automaticamente `terraform.tfvars` y cualquier archivo con el patron `*.auto.tfvars` en el directorio de trabajo. Para entornos especificos se usa `-var-file`:

```bash
terraform apply -var-file="prod.tfvars"
terraform apply -var-file="dev.tfvars"
```

### Parametro -var en CLI

Util para asignaciones puntuales o en scripts de automatizacion:

```bash
terraform apply -var="instancias=5" -var='tags={Proyecto="Web"}'
```

### Variables de entorno TF_VAR_

Cualquier variable de entorno con el prefijo `TF_VAR_` se asigna automaticamente a la variable correspondiente:

```bash
export TF_VAR_region_aws=us-west-2
export TF_VAR_instancias=3
terraform apply
```

Este mecanismo es especialmente util en pipelines de CI/CD donde las credenciales y configuraciones se inyectan mediante variables de entorno del sistema.

> Las variables de entorno se evaluan como strings. Si una variable espera un tipo `number`, `bool`, `list` o `map`, Terraform intentara parsear el string. Para tipos complejos, se recomienda usar archivos `.tfvars` o `-var` con la sintaxis completa.

## Validaciones

El bloque `validation` permite definir reglas que los valores asignados deben cumplir. Las validaciones se evaluan durante `terraform plan` y `terraform validate`, antes de cualquier interaccion con los proveedores.

```hcl
variable "region_aws" {
  description = "Region de AWS"
  type        = string

  validation {
    condition     = contains(["us-east-1", "us-west-2", "eu-west-1"], var.region_aws)
    error_message = "La region debe ser us-east-1, us-west-2 o eu-west-1."
  }
}

variable "instancias" {
  description = "Numero de instancias"
  type        = number

  validation {
    condition     = var.instancias >= 1 && var.instancias <= 10
    error_message = "El numero de instancias debe estar entre 1 y 10."
  }
}

variable "nombre_proyecto" {
  description = "Nombre del proyecto"
  type        = string

  validation {
    condition     = length(var.nombre_proyecto) > 3 && can(regex("^[a-zA-Z0-9_-]+$", var.nombre_proyecto))
    error_message = "El nombre debe tener al menos 3 caracteres y solo contener letras, numeros, guiones y guiones bajos."
  }
}
```

La funcion `can()` en una validacion evalua una expresion y devuelve `true` si no produce error. Es util para comprobar tipos o estructuras de datos:

```hcl
variable "config_red" {
  type = object({
    vpc_cidr     = string
    enable_nat   = bool
  })

  validation {
    condition = can(cidrhost(var.config_red.vpc_cidr, 0))
    error_message = "El CIDR de la VPC no tiene un formato valido."
  }
}
```

> Las validaciones solo se ejecutan si la variable recibe un valor. Si una variable tiene un `default` y no se sobrescribe, el valor por defecto tambien se valida. Esto implica que cambiar una validacion puede romper configuraciones existentes si el valor por defecto no cumple la nueva regla.

## Expresiones condicionales con variables

Las variables permiten construir logica condicional mediante el operador ternario `condicion ? valor_true : valor_false`. Este patron es omnipresente en configuraciones Terraform para adaptar el comportamiento segun el entorno.

```hcl
resource "aws_instance" "servidor" {
  ami           = var.ami_id
  instance_type = var.entorno == "produccion" ? "t3.large" : "t2.micro"
  count         = var.entorno == "produccion" ? 3 : 1

  tags = {
    Name     = "servidor-${var.entorno}"
    Backups  = var.entorno == "produccion" ? "activados" : "desactivados"
  }
}
```

Para condicionales mas complejos, las variables de tipo `map` pueden actuar como tablas de decision:

```hcl
variable "tipo_instancia_por_entorno" {
  type = map(string)
  default = {
    desarrollo   = "t2.micro"
    pruebas      = "t3.small"
    produccion   = "t3.large"
  }
}

resource "aws_instance" "servidor" {
  instance_type = var.tipo_instancia_por_entorno[var.entorno]
}
```

## Valores locales

Ademas de las variables de entrada, Terraform permite definir valores locales con el bloque `locals`. Los locales son expresiones calculadas que simplifican las referencias y evitan la repeticion de logica.

```hcl
locals {
  nombre_completo = "${var.proyecto}-${var.entorno}"
  tags_ comunes = {
    Proyecto  = var.proyecto
    Entorno   = var.entorno
    Gestion   = "Terraform"
  }
  es_produccion = var.entorno == "produccion"
}

resource "aws_instance" "servidor" {
  tags = merge(local.tags_comunes, {
    Name = local.nombre_completo
  })
}
```

Los valores locales se referencian con `local.<nombre>` y se evaluan una sola vez durante el plan. No pueden definirse desde fuera de la configuracion, a diferencia de las variables de entrada.

