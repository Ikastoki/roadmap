# Estructura Estandar de un Modulo

La previsibilidad es un activo critico en infraestructura como codigo. Cuando un equipo trabaja con docenas de modulos provenientes de diferentes autores, la ausencia de convenciones estructurales genera una friccion constante: cada modulo requiere aprenderse de cero, la localizacion de configuraciones se vuelve arbitraria, y las herramientas de automatizacion pierden eficacia al no poder asumir ubicaciones estandar para archivos criticos. La estructura estandar de un modulo resuelve estos problemas estableciendo un contrato implicito sobre como se organizan las definiciones, las interfaces y la documentacion.

Un modulo Terraform sigue una convencion ampliamente adoptada que separa responsabilidades en archivos distintos, cada uno con un proposito claro. Esta separacion no es caprichosa: permite que herramientas de linting, validacion y generacion automatica de documentacion funcionen sin configuracion adicional, y que cualquier desarrollador experimentado pueda navegar el modulo sin instrucciones previas.

## Estructura de Directorios

La disposicion fisica de un modulo en el sistema de archivos sigue un patron consistente:

```
modules/
└── vpc/
    ├── main.tf          # Definiciones de recursos principales
    ├── variables.tf     # Interfaz de entrada (parametros)
    ├── outputs.tf       # Interfaz de salida (valores exportados)
    ├── versions.tf      # Restricciones de versiones de proveedores
    ├── README.md        # Documentacion del modulo
    └── examples/
        └── complete/
            ├── main.tf
            └── outputs.tf
```

Cada archivo tiene una responsabilidad unica. `main.tf` contiene la logica principal: los recursos que el modulo crea, modifica o gestiona. `variables.tf` define formalmente que parametros acepta el modulo, con tipos, valores por defecto y validaciones. `outputs.tf` expone los valores que otros modulos o configuraciones pueden consumir. `versions.tf` fija las versiones minimas de Terraform y de cada proveedor utilizado.

## Definicion de Variables: la Interfaz de Entrada

Las variables constituyen el contrato entre el modulo y sus consumidores. Una variable mal definida —sin tipo, sin validacion, sin descripcion— genera ambiguedad y errores en tiempo de ejecucion que podrian haberse detectado en compilacion:

```hcl
# variables.tf

variable "name" {
  description = "Nombre base para todos los recursos creados por este modulo"
  type        = string

  validation {
    condition     = length(var.name) > 0 && length(var.name) <= 64
    error_message = "El nombre debe tener entre 1 y 64 caracteres"
  }
}

variable "vpc_id" {
  description = "ID de la VPC donde se desplegaran los recursos"
  type        = string

  validation {
    condition     = can(regex("^vpc-[a-f0-9]{8,}$", var.vpc_id))
    error_message = "El vpc_id debe ser un ID valido de VPC de AWS"
  }
}

variable "instance_type" {
  description = "Tipo de instancia EC2 a utilizar"
  type        = string
  default     = "t3.micro"

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Solo se permiten instancias de la familia t2 o t3"
  }
}

variable "tags" {
  description = "Mapa de etiquetas a aplicar a todos los recursos"
  type        = map(string)
  default     = {}
}

variable "enable_monitoring" {
  description = "Habilitar metricas detalladas de CloudWatch"
  type        = bool
  default     = false
}
```

Las validaciones en `condition` interceptan valores invalidos antes de que Terraform intente crear recursos. Sin ellas, un `vpc_id` mal formado produciria un error del proveedor AWS, mucho menos informativo y mas costoso de diagnosticar.

> Las variables sin valores por defecto son obligatorias. Terraform exigira al consumidor que las proporcione explicitamente, lo cual es apropiado para valores criticos como identificadores de red o nombres de dominio que nunca deben asumirse.

## Definicion de Outputs: la Interfaz de Salida

Los outputs son la segunda mitad del contrato del modulo. Exponen valores que otros componentes del sistema necesitan —IDs de recursos, direcciones IP, nombres de dominio— y ocultan todo lo demas:

```hcl
# outputs.tf

output "instance_id" {
  description = "ID de la instancia EC2 creada"
  value       = aws_instance.main.id
}

output "public_ip" {
  description = "Direccion IP publica de la instancia"
  value       = aws_instance.main.public_ip
}

output "private_dns" {
  description = "DNS privado de la instancia dentro de la VPC"
  value       = aws_instance.main.private_dns
}

output "security_group_id" {
  description = "ID del grupo de seguridad creado para la instancia"
  value       = aws_security_group.instance.id
}

output "arn" {
  description = "ARN completo de la instancia EC2"
  value       = aws_instance.main.arn
}
```

El consumidor accede a estos valores mediante la sintaxis `module.<nombre_modulo>.<output>`. Esta referencia es la unica forma en que los modulos comparten informacion entre si, manteniendo un acoplamiento minimo:

```hcl
module "app_server" {
  source = "./modules/ec2-instance"

  name          = "app-primary"
  vpc_id        = module.networking.vpc_id
  instance_type = "t3.medium"
}

module "load_balancer" {
  source = "./modules/alb"

  name              = "app-lb"
  subnet_ids        = module.networking.public_subnet_ids
  target_instance_id = module.app_server.instance_id
}
```

## Versionado del Proveedor y de Terraform

`versions.tf` establece las restricciones de compatibilidad que garantizan reproducibilidad. Sin esta restriccion, una actualizacion automatica de un proveedor puede introducir cambios incompatibles que rompan la infraestructura existente:

```hcl
# versions.tf

terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}
```

La restriccion `~> 5.0` permite actualizaciones de parche y menores (5.1, 5.2) pero bloquea versiones mayores (6.0). Esto balancea seguridad —obtener correcciones de bugs— con estabilidad —evitar cambios de API.

## Documentacion y Ejemplos

El `README.md` de un modulo debe incluir al menos: una descripcion del proposito, una tabla de variables con tipos y valores por defecto, una tabla de outputs, y un ejemplo de uso basico. Las herramientas como `terraform-docs` generan esta documentacion automaticamente desde los archivos fuente:

```bash
terraform-docs markdown table ./modules/vpc > ./modules/vpc/README.md
```

La carpeta `examples/` contiene configuraciones completas y funcionales que demuestran el uso del modulo en contexto. Estos ejemplos sirven simultaneamente como documentacion viviente y como tests de integracion: si el ejemplo no aplica, el modulo tiene un problema de diseno o documentacion.

> Un modulo sin documentacion es un modulo incompleto. La documentacion no es un adorno —es parte de la interfaz publica del modulo y determina si otros equipos lo adoptaran o lo rechazaran.

