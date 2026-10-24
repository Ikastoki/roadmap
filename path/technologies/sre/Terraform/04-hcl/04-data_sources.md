# Data Sources

Los data sources permiten consultar informacion de infraestructura existente que no ha sido creada por Terraform. Mientras que los recursos (`resource`) definen infraestructura que Terraform crea, modifica y destruye, los data sources son unicamente de lectura: obtienen datos de un proveedor y los ponen a disposicion de la configuracion sin realizar cambios en el mundo real.

Esta capacidad es esencial para integrar Terraform con entornos preexistentes, recuperar identificadores de recursos gestionados fuera de Terraform, o simplemente obtener informacion contextual del proveedor (como las zonas de disponibilidad disponibles en una region).

## Sintaxis basica

La estructura de un data source sigue el mismo patron de bloques que los recursos, pero utilizando la palabra clave `data` en lugar de `resource`:

```hcl
data "tipo" "nombre_local" {
  filtro_1 = "valor_1"
  filtro_2 = "valor_2"
}
```

El `tipo` identifica el data source dentro del proveedor (por ejemplo, `aws_ami`, `aws_vpc`, `azurerm_resource_group`). El `nombre_local` es un identificador unico para referenciar los datos dentro de la configuracion.

Para acceder a los atributos de un data source se utiliza la sintaxis `data.<tipo>.<nombre_local>.<atributo>`:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

output "ubuntu_ami_id" {
  value = data.aws_ami.ubuntu.id
}
```

En este ejemplo, Terraform consulta a AWS por la AMI de Ubuntu 22.04 mas reciente que cumpla los filtros especificados. No crea ningun recurso; simplemente recupera informacion que puede usarse para crear instancias u otros recursos.

> Los data sources se evaluan durante `terraform plan` y su valor queda registrado en el estado. Si los datos consultados cambian en el mundo real, Terraform no lo detectara hasta el proximo `plan` o `apply`.

## Ejemplos comunes

### Zonas de disponibilidad

```hcl
data "aws_availability_zones" "disponibles" {
  state = "available"
}

resource "aws_subnet" "principal" {
  count             = length(data.aws_availability_zones.disponibles.names)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.disponibles.names[count.index]
}
```

Este patron permite desplegar subnets en todas las AZs disponibles sin necesidad de conocerlas de antemano. Si una region tiene 3 AZs, se crearan 3 subnets. Si otra tiene 2, se crearan 2.

### VPC existente

```hcl
data "aws_vpc" "existente" {
  tags = {
    Name = "vpc-produccion"
  }
}

data "aws_subnet_ids" "subnets_existentes" {
  vpc_id = data.aws_vpc.existente.id
}

resource "aws_instance" "web" {
  subnet_id     = tolist(data.aws_subnet_ids.subnets_existentes.ids)[0]
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}
```

### Imagen de contenedor ECR

```hcl
data "aws_ecr_repository" "app" {
  name = "mi-aplicacion"
}

data "aws_ecr_image" "ultima" {
  repository_name = data.aws_ecr_repository.app.name
  image_tag       = "latest"
}

resource "aws_ecs_task_definition" "app" {
  family = "servicio-app"
  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${data.aws_ecr_repository.app.repository_url}:${data.aws_ecr_image.ultima.image_tag}"
      memory = 512
    }
  ])
}
```

## Diferencia entre data sources y recursos

La distincion fundamental es que `resource` declara infraestructura que Terraform gestiona activamente (crea, modifica, destruye), mientras que `data` solo consulta datos existentes. Sin embargo, los data sources pueden tener efectos secundarios en ciertos proveedores. Por ejemplo, el data source `aws_secretsmanager_secret_version` no modifica nada, pero `aws_caller_identity` simplemente devuelve informacion de la cuenta.

| Aspecto | resource | data |
|---------|----------|------|
| Modifica infraestructura | Si | No |
| Aparece en `terraform destroy` | Si | No |
| Puede tener `count`/`for_each` | Si | Si |
| Soporta `depends_on` | Si | Si |
| Se almacena en estado | Si | Si |

## Data sources con for_each y count

Al igual que los recursos, los data sources pueden utilizar `count` y `for_each` para consultar multiples conjuntos de datos:

```hcl
variable "nombres_amis" {
  type    = list(string)
  default = ["ubuntu", "amazon-linux", "redhat"]
}

data "aws_ami" "imagenes" {
  for_each = toset(var.nombres_amis)

  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["${each.key}-*"]
  }
}

output "amis_disponibles" {
  value = {
    for nombre, ami in data.aws_ami.imagenes :
    nombre => ami.id
  }
}
```

## Data sources locales

Ademas de los data sources de proveedores, Terraform dispone de data sources integrados que no requieren proveedor:

```hcl
# Informacion del template actual (ruta, directorio)
data "terraform_remote_state" "backend" {
  backend = "s3"
  config = {
    bucket = "mi-bucket-estado"
    key    = "infraestructura-base/terraform.tfstate"
    region = "us-east-1"
  }
}
```

El data source `terraform_remote_state` permite leer el estado de otra configuracion de Terraform. Es util para infraestructuras divididas en capas (red, seguridad, aplicacion) donde cada capa tiene su propio estado.

```hcl
# Acceder a outputs de otra configuracion
vpc_id = data.terraform_remote_state.base.outputs.vpc_id
```

## Filtrado y busqueda

Los data sources suelen ofrecer multiples criterios de filtrado. La combinacion de filtros determina que datos se recuperan:

```hcl
data "aws_security_group" "existente" {
  filter {
    name   = "group-name"
    values = ["sg-acceso-publico"]
  }

  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existente.id]
  }
}
```

Algunos data sources devuelven un unico recurso (el primero que coincide), mientras que otros devuelven colecciones. La opcion `most_recent = true` en `aws_ami` selecciona la imagen mas reciente entre las que coinciden con los filtros.

> Si un data source con filtros no encuentra coincidencias, Terraform falla con un error. No se puede usar un data source para comprobar condicionalmente si un recurso existe. Para esa logica se necesitarian metodos externos o scripts previos a Terraform.

