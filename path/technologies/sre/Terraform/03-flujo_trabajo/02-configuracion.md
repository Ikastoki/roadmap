# Escritura de configuracion

La configuracion en Terraform se expresa mediante bloques escritos en HCL (HashiCorp Configuration Language), un lenguaje declarativo disenado para describir infraestructura de forma legible y estructurada. Cada archivo `.tf` contiene una serie de bloques que definen tanto la configuracion global del proyecto como los recursos concretos que se desean provisionar. Comprender la funcion y la sintaxis de cada tipo de bloque es esencial para construir configuraciones correctas y mantenibles.

## Bloque terraform

El bloque `terraform` es el punto de entrada de cualquier configuracion. Contiene las ajustes globales del proyecto, incluyendo las versiones de Terraform requeridas, los providers necesarios y la configuracion del backend donde se almacenara el estado. Aunque es tecnicamente opcional, en la practica deberia estar siempre presente para garantizar la reproducibilidad del proyecto.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "3.5.1"
    }
  }

  backend "s3" {
    bucket = "mi-estado-terraform"
    key    = "proyecto/terraform.tfstate"
    region = "us-east-1"
  }
}
```

El parametro `required_version` restringe la version del binario de Terraform que puede ejecutar esta configuracion. Esto evita que alguien aplique cambios con una version incompatible que podria interpretar el HCL de forma diferente. El bloque `required_providers` declara que providers se necesitan y de donde descargarlos. La clave `source` sigue el formato `namespace/tipo` (por ejemplo, `hashicorp/aws`), y `version` admite cualquier restriccion de version compatible con el operador utilizado.

## Bloque provider

El bloque `provider` configura la instancia concreta de un provider que Terraform utilizara para interactuar con el servicio. Mientras que `required_providers` declara que plugin descargar, el bloque `provider` establece los parametros de conexion: region, credenciales, endpoint personalizado, etc.

```hcl
provider "aws" {
  region = "eu-west-1"

  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/terraform-role"
  }

  default_tags {
    tags = {
      Proyecto = "Infraestructura"
      Gestion  = "Terraform"
    }
  }
}
```

Se pueden definir multiples bloques del mismo provider utilizando el argumento `alias`, lo que permite gestionar recursos en varias regiones o cuentas desde la misma configuracion.

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

Para utilizar un provider con alias, los recursos deben referenciarlo explicitamente mediante el argumento `provider`:

```hcl
resource "aws_instance" "principal" {
  # Usa el provider por defecto (us-east-1)
  ami = "ami-0c55b159cbfafe1f0"
}

resource "aws_instance" "secundario" {
  provider = aws.west
  ami      = "ami-0c55b159cbfafe1f0"
}
```

> Las credenciales nunca deben incluirse directamente en los archivos de configuracion. Terraform las obtiene automaticamente de las variables de entorno (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`), de los archivos de credenciales de AWS o de roles IAM en instancias EC2.

## Bloque resource

Los bloques `resource` son el corazon de Terraform: describen los componentes de infraestructura que se desean crear, modificar o eliminar. Cada recurso pertenece a un provider y tiene un tipo especifico que determina que parametros acepta y que comportamiento tiene.

```hcl
resource "aws_instance" "servidor_web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  subnet_id = aws_subnet.publica.id

  tags = {
    Name = "ServidorWeb"
  }
}
```

La sintaxis de un bloque resource sigue el patron:

```
resource "tipo_recurso" "nombre_local" {
  argumento1 = valor1
  argumento2 = valor2
}
```

El tipo de recurso (`aws_instance`, `aws_s3_bucket`, `random_pet`) determina que argumentos son obligatorios y cuales opcionales. El nombre local es un identificador unico dentro del modulo que permite referenciar el recurso desde otras partes de la configuracion mediante la sintaxis `tipo_recurso.nombre_local.atributo`.

Los recursos pueden depender de otros recursos de forma implicita o explicita. La dependencia implicita se produce cuando un recurso referencia un atributo de otro recurso, como en `subnet_id = aws_subnet.publica.id`. Terraform analiza estas referencias y construye automaticamente el grafo de dependencias para determinar el orden de creacion.

## Ejemplo basico completo

Un proyecto minimo pero funcional que crea una instancia EC2 con un grupo de seguridad:

```hcl
# main.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_security_group" "web" {
  name_prefix = "web-"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name = "ServidorWeb"
  }
}
```

> El orden de los bloques dentro del archivo no afecta al comportamiento. Terraform analiza todos los archivos `.tf` del directorio y construye el grafo de dependencias completo antes de ejecutar cualquier operacion. Esto permite organizar los bloques de la forma que resulte mas legible para el equipo.

