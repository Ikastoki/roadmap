# Conceptos Clave de IaC

Antes de profundizar en ciclos de vida, modulos avanzados o flujos de CI/CD, es necesario establecer un vocabulario comun. Los conceptos que siguen aparecen en todas las herramientas de Infraestructura como Codigo, independientemente del lenguaje o proveedor. Dominarlos permite comunicarse con precision y comprender documentacion, errores de ejecucion y patrones de arquitectura sin ambiguedades.

## Resource (Recurso)

Un recurso es la unidad basica de infraestructura que una herramienta IaC puede crear, modificar o eliminar. Representa un objeto concreto en el mundo real: una maquina virtual, una base de datos, un balanceador de carga, un registro DNS o un bucket de almacenamiento. Cada recurso tiene un tipo que define que propiedades soporta, y un nombre local que lo identifica dentro del proyecto.

```hcl
# Recurso basico - instancia EC2
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id

  root_block_device {
    volume_size = 30
    volume_type = "gp3"
  }

  tags = {
    Name = "web-server"
  }
}

# Recurso dependiente - grupo de seguridad
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

> Los recursos nunca se crean de forma aislada. El motor construye un grafo de dependencias basado en las referencias entre recursos y los ejecuta en el orden correcto. En el ejemplo anterior, `aws_security_group.web` se crea antes que `aws_instance.web_server` porque esta referenciada en `vpc_security_group_ids`.

## Module / Componente

Un modulo es un paquete reutilizable que encapsula un conjunto de recursos logicamente agrupados. En lugar de definir toda la infraestructura en un solo fichero, los modulos permiten abstracter patrones comunes (una VPC completa, un cluster Kubernetes, una base de datos con replicas) y reutilizarlos en multiples proyectos o entornos. Expos variables para parametrizar su comportamiento y outputs para exportar valores relevantes.

```hcl
# Definicion de un modulo local
module "database" {
  source = "./modules/database"

  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r6g.large"
  allocated_storage = 100

  vpc_id          = aws_vpc.main.id
  subnet_ids      = aws_subnet.private[*].id
  security_groups = [aws_security_group.db.id]

  backup_retention_days = 7
  multi_az              = true
}

# Uso del output del modulo
resource "aws_application" "api" {
  # ...
  database_endpoint = module.database.endpoint
}
```

```hcl
# Internamente, el modulo database/ contiene:
# modules/database/
# ├── main.tf        # Recursos: db_instance, db_subnet_group, etc.
# ├── variables.tf   # Variables de entrada del modulo
# └── outputs.tf     # Valores exportados (endpoint, port, etc.)
#
# variables.tf:
#   variable "engine" { type = string }
#   variable "engine_version" { type = string }
#   variable "instance_class" { type = string }
#   ...
#
# outputs.tf:
#   output "endpoint" {
#     value = aws_db_instance.main.endpoint
#   }
#   output "port" {
#     value = aws_db_instance.main.port
#   }
```

> Un modulo no es un recurso ni un provider. Es una forma de organizar y reutilizar codigo. Terraform permite combinar modulos de fuentes locales (directorio), registros publicos (Terraform Registry) y repositorios Git, todos con la misma sintaxis de invocacion.

## Stack / Proyecto

Un stack es el conjunto completo de recursos que se despliegan juntos como una unidad coherente. En la practica, un stack corresponde a un directorio con su propio estado, lo que permite gestionar entornos de forma independiente (dev, staging, produccion) sin que un cambio en uno afecte a los demas. El concepto de stack no esta formalizado en Terraform, pero es un patron universal en todas las herramientas IaC.

```bash
# Estructura tipica de stacks por entorno
infrastructure/
├── modules/                 # Modulos compartidos
│   ├── vpc/
│   ├── eks/
│   └── rds/
├── envs/
│   ├── dev/                 # Stack de desarrollo
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars # Valores especificos de dev
│   ├── staging/             # Stack de staging
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   └── prod/                # Stack de produccion
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
```

```bash
# Despliegue de stacks independientes
cd envs/dev && terraform apply    # Solo afecta dev
cd envs/prod && terraform apply  # Solo afecta prod
```

> Mantener stacks separados tiene un costo: duplicacion de archivos `main.tf` (a menos que se use un modulo unico con `for_each` sobre entornos). La alternativa es usar workspaces, pero estos comparten estado y no ofrecen aislamiento real. Para entornos con diferencias significativas, stacks separados es la practica recomendada.

## Variable

Las variables son parametros de entrada que parametrizan el codigo, eliminando valores hardcodeados y permitiendo que un mismo modulo sirva para multiples escenarios. Terraform soporta multiples tipos (string, number, bool, list, map, object) y permite establecer valores por defecto, validaciones y descripciones.

```hcl
# Declaracion de variables
variable "environment" {
  description = "Nombre del entorno (dev, staging, prod)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "El entorno debe ser dev, staging o prod."
  }
}

variable "instance_count" {
  description = "Numero de instancias a crear"
  type        = number
  default     = 2
}

variable "tags" {
  description = "Tags comunes para todos los recursos"
  type        = map(string)
  default = {
    ManagedBy = "terraform"
  }
}

variable "enable_monitoring" {
  description = "Habilitar metricas de CloudWatch"
  type        = bool
  default     = true
}
```

```bash
# Formas de inyectar valores a las variables
# 1. Archivo .tfvars (recomendado)
terraform apply -var-file="produccion.tfvars"

# 2. Variable de entorno
export TF_VAR_environment=prod
terraform apply

# 3. Flag en linea de comandos
terraform apply -var="environment=prod"

# 4. terraform.tfvars en el directorio del proyecto
echo 'environment = "dev"' > terraform.tfvars
```

> Las variables marcadas como `sensitive = true` no se muestran en el output de plan o apply, pero se almacenan en texto plano en el estado. Para datos verdaderamente secretos (credenciales, certificados) es preferible usar Vault, SSM Parameter Store o secrets de CI/CD en lugar de variables de Terraform.

## Output (Salida)

Los outputs son valores exportados que el modulo o proyecto pone a disposicion de otros componentes. Permiten exponer informacion generada por los recursos (IPs publicos, ARNs, endpoints de bases de datos) sin exponer toda la estructura interna del estado. Son la interfaz de comunicacion entre modulos y entre stacks.

```hcl
# Definicion de outputs
output "vpc_id" {
  description = "ID de la VPC creada"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs de las subnets publicas"
  value       = aws_subnet.public[*].id
}

output "load_balancer_dns" {
  description = "DNS del balanceador de carga"
  value       = aws_lb.main.dns_name
}

output "database_endpoint" {
  description = "Endpoint de la base de datos"
  value       = aws_db_instance.main.endpoint
  sensitive   = true  # No se muestra en la CLI
}
```

```hcl
# Consumo de outputs de un modulo en otro
module "networking" {
  source = "./modules/networking"
  # ...
}

module "compute" {
  source = "./modules/compute"

  vpc_id     = module.networking.vpc_id          # Output referenciado
  subnet_ids = module.networking.public_subnet_ids
}

# Output del modulo padre que re-expone un valor del modulo hijo
output "api_endpoint" {
  value = module.compute.load_balancer_dns
}
```

> Los outputs no son variables. Las variables son entradas; los outputs son salidas. Un modulo no puede leer los outputs de otro modulo directamente; solo puede acceder a ellos a traves de la referencia explicita `module.<nombre_modulo>.<nombre_output>`.

## Data Source (Consulta de Datos)

Un data source permite a la herramienta IaC consultar recursos que existen pero que no estan gestionados por ella. Esto es util para obtener IDs de AMIs actualizadas, listar subnets existentes o resolver nombres de dominio sin tener que importar esos recursos al estado. Los data sources son de solo lectura: nunca crean, modifican o eliminan recursos.

```hcl
# Consulta de la AMI mas reciente de Amazon Linux 2023
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Consulta de todas las subnets de una VPC existente
data "aws_subnets" "existing" {
  filter {
    name   = "vpc-id"
    values = ["vpc-0abc123def456"]
  }

  tags = {
    Environment = "production"
  }
}

# Uso del data source en un recurso
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id  # ID dinamico
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnets.existing.ids[0]
}
```

```hcl
# Data source para resolver un dominio DNS
data "dns_a_record_set" "api_gateway" {
  host = "api.example.com"
}

# Data source para obtener caller identity de AWS
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

> Los data sources se resuelven durante la fase de plan. Si el recurso consultado no existe o la consulta no retorna resultados, Terraform detiene la ejecucion con un error. Esto los hace utiles como mecanismos de validacion implicita: si la AMI no existe, el despliegue falla antes de intentar crear la instancia.

