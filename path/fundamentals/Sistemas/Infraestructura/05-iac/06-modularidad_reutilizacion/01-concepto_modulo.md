# Concepto de Modulo en Infraestructura como Codigo

La infraestructura como codigo moderna se construye sobre el principio de composicion: sistemas complejos derivados de unidades simples, testeadas y reutilizables. Un modulo encapsula un conjunto de recursos relacionados dentro de una unidad logica con una interfaz bien definida, permitiendo que la misma configuracion se aplique en distintos entornos sin duplicar codigo. Este patron resuelve uno de los problemas mas persistentes en la gestion de infraestructura: la divergencia entre entornos相似 al copiar y pegar configuraciones entre carpetas o repositorios.

La encapsulacion que ofrece un modulo no es meramente organizativa. El modulo actua como una frontera de abstraccion donde los recursos internos quedan ocultos al consumidor, quien solo interactua mediante variables de entrada y valores de salida. Esta separacion permite cambiar la implementacion interna —migrar de un proveedor a otro, optimizar configuraciones, agregar recursos auxiliares— sin afectar a los consumidores del modulo, siempre que la interfaz se mantenga compatible. El resultado es una infraestructura mas estable, mas facil de mantener y con un superficie de error significativamente reducida.

## Abstraccion de Complejidad

Un servidor web en produccion no es solo una instancia EC2 o una VM en Azure. Implica subredes, grupos de seguridad, registros DNS, politicas IAM, certificados TLS, configuraciones de monitoreo y reglas de firewall. Sin modulos, cada uno de estos recursos debe definirse y referenciarse explicitamente en el archivo principal, creando configuraciones de cientos de lineas donde un cambio accidental puede producir interrupciones.

Un modulo oculta esta complejidad detras de una interfaz minimalista. El consumidor declara que necesita un "servidor web" y proporciona los parametros esenciales; el modulo se encarga de orquestar todos los recursos subyacentes:

```hcl
module "web_server" {
  source = "./modules/web-server"

  instance_type = "t3.medium"
  vpc_id        = module.vpc.vpc_id
  subnet_id     = module.vpc.public_subnet_ids[0]
  domain_name   = "api.ejemplo.com"
  environment   = "production"
}
```

Este fragmento sustituye la definicion manual de diez o mas recursos de Terraform. La complejidad de configurar HTTPS, security groups con puertos exactos, y associacion de IPs elasticas queda resuelta internamente por el modulo.

> Un modulo bien disenado es aquel cuyo nombre describe exactamente que crea, sin necesidad de inspeccionar su codigo interno. Si el consumidor debe leer `main.tf` para entender que hace el modulo, la abstraccion es insuficiente.

## Principio DRY aplicado a Infraestructura

DRY — Don't Repeat Yourself — se traduce en infraestructura como codigo como la eliminacion sistematica de definiciones duplicadas. Sin modulos, es comun encontrar patrones como este en multiples archivos de configuracion:

```hcl
# duplicado en dev/main.tf
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = {
    Name        = "app-dev"
    Environment = "dev"
  }
}

# duplicado en staging/main.tf  
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"
  tags = {
    Name        = "app-staging"
    Environment = "staging"
  }
}
```

Cada modificacion —actualizar la AMI, agregar una etiqueta, cambiar un atributo— debe realizarse en cada ubicacion donde existe la duplicacion. El modulo elimina esta fuente de errores:

```hcl
module "app_instance" {
  source = "./modules/ec2-instance"

  for_each = {
    dev     = { instance_type = "t3.micro",  count = 1 }
    staging = { instance_type = "t3.small",  count = 2 }
  }

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value.instance_type
  name_prefix   = "app-${each.key}"
  environment   = each.key
}
```

Una sola definicion, multiples instancias parametrizadas. La actualizacion de la AMI ocurre en un unico lugar.

## Niveles de Abstraccion

Los modulos pueden componerse entre si, creando jerarquias de abstraccion que reflejan la arquitectura del sistema. Un modulo de "base de datos" puede internamente invocar modulos de "subnet", "grupo de seguridad" y "grupo de parametros de RDS":

```hcl
# modulo interno: modules/database/main.tf

module "db_subnet_group" {
  source = "../network/subnet-group"

  name       = "${var.name}-db-subnets"
  subnet_ids = var.subnet_ids
}

module "security_group" {
  source = "../security/sg-with-rules"

  name        = "${var.name}-db-sg"
  vpc_id      = var.vpc_id
  ingress_rules = [
    { port = 5432, source = var.app_security_group_id }
  ]
}

resource "aws_db_instance" "main" {
  identifier     = var.name
  engine         = var.engine
  instance_class = var.instance_class
  
  db_subnet_group_name   = module.db_subnet_group.name
  vpc_security_group_ids = [module.security_group.id]
  
  # ... demas configuraciones
}
```

Esta composicion permite construir infraestructura compleja a partir de piezas simples y testeadas. Cada nivel de la jerarquia encapsula solo la complejidad que le corresponde, y el consumidor final interactua exclusivamente con el modulo de mas alto nivel.

> La composicion de modulos es análoga a las funciones en programacion: una funcion puede llamar a otras funciones, cada una realizando una tarea especifica. La complejidad total del sistema se gestiona a traves de la jerarquia de abstracciones, no mediante archivos monoliticos.

