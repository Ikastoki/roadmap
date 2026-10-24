# Dependencias implicitas y explicitas

Terraform construye internamente un grafo aciclico dirigido (DAG) que determina el orden en que los recursos deben crearse, modificarse o destruirse. La mayoria de las aristas de ese grafo se generan automaticamente a partir de las referencias entre atributos de recursos, pero hay casos donde Terraform no puede inferir la relacion y es necesario declararla manualmente. Entender esta distincion es clave para escribir configuraciones predecibles y evitar errores en entornos complejos.

## Dependencias implicitas

Cuando un recurso referencia un atributo de otro recurso, Terraform reconoce automaticamente que existe una relacion de dependencia. Por ejemplo, al asignar `subnet_id = aws_subnet.privada.id` dentro de una instancia EC2, Terraform sabe que la subred debe existir antes que la instancia, y que la instancia debe destruirse antes que la subred si se modifica algo.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "privada" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "app" {
  subnet_id = aws_subnet.privada.id
  ami       = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

La referencia `aws_vpc.main.id` dentro del recurso `aws_subnet.privada` crea una arista en el grafo que asegura el orden correcto. Terraform analiza todas estas referencias durante la fase de planificacion y las materializa en el plan de ejecucion. Si no hay dependencias entre dos recursos, Terraform los crea en paralelo para maximizar el rendimiento.

```hcl
resource "aws_security_group" "web" {
  name = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

En este fragmento, `aws_security_group.web` depende implicitamente de `aws_vpc.main` por la referencia directa a `vpc_id`. Si se eliminara la referencia pero el security group siguiera necesitando que la VPC exista, la dependencia dejaria de ser automatica.

> Las dependencias implicitas tambien pueden propagarse a traves de modulos y data sources. Si un modulo hijo expone un output que referencia un recurso interno, cualquier recurso que consuma ese output en el modulo padre hereda la dependencia de forma transitiva.

## Dependencias explicitas con depends_on

Hay escenarios donde Terraform no puede inferir una dependencia porque no existe una referencia directa de atributo entre recursos, pero la relacion existe a nivel de comportamiento o de efectos secundarios. El ejemplo clasico son los provisioners: si un recurso local-exec ejecuta un comando que requiere que un balanceador este operativo, Terraform no lo sabe a menos que se le indique.

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  provisioner "local-exec" {
    command = "echo ${aws_lb.web.dns_name} > /tmp/lb_dns.txt"
  }
}

resource "aws_lb" "web" {
  name = "web-lb"
  # ...
}
```

Aqui `aws_instance.app` no referencia ningun atributo de `aws_lb.web` en su configuracion, solo en el provisioner. Terraform no contabiliza las referencias dentro de provisioners como dependencias implicitas, por lo que podria crear la instancia antes que el balanceador y el comando fallaria. La solucion es usar `depends_on`:

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  depends_on = [
    aws_lb.web
  ]

  provisioner "local-exec" {
    command = "echo ${aws_lb.web.dns_name} > /tmp/lb_dns.txt"
  }
}
```

`depends_on` acepta una lista de referencias a recursos o modulos. Le dice a Terraform: "este recurso depende de estos otros, aunque no haya referencias directas de datos". El argumento esta disponible en cualquier bloque `resource` o `module`.

```hcl
module "network" {
  source = "./modules/network"
}

module "app" {
  source = "./modules/app"

  depends_on = [module.network]
}
```

> `depends_on` no debe usarse de forma preventiva. Anadir dependencias innecesarias reduce la capacidad de Terraform para paralelizar la creacion de recursos y ralentiza las ejecuciones. Solo debe emplearse cuando la relacion no pueda expresarse mediante una referencia de atributo.

## Casos de uso tipicos

Las dependencias explicitas son necesarias cuando un recurso depende de una accion que no genera un valor consultable, como la ejecucion de un script en un provisioner `remote-exec`, la propagacion de politicas IAM que tardan segundos en surtir efecto, o la finalizacion de un `null_resource` que ejecuta un comando externo. Tambien son frecuentes en configuraciones donde un recurso depende de la existencia de otro, pero no necesita ninguno de sus atributos directamente.

```hcl
resource "null_resource" "inicializar_base_datos" {
  provisioner "local-exec" {
    command = "curl -X POST https://db-internal.example.com/setup"
  }
}

resource "aws_instance" "app" {
  # Depende de que la base de datos este lista,
  # aunque no referencie ningun atributo de null_resource
  depends_on = [null_resource.inicializar_base_datos]
}
```

El uso correcto de dependencias implicitas y explicitas determina la velocidad y fiabilidad de las ejecuciones de Terraform. El principio general es: expresar toda relacion de orden mediante referencias de atributo siempre que sea posible, y reservar `depends_on` para los casos donde la relacion es real pero no capturable con referencias directas.

