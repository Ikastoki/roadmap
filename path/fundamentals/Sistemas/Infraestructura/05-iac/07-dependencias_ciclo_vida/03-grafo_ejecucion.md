# Grafo de ejecucion

Terraform no ejecuta las operaciones sobre los recursos de forma secuencial ni aleatoria. En su lugar, construye internamente un **grafo aciclico dirigido** (DAG, por sus siglas en ingles) que representa todas las dependencias entre los recursos definidos en la configuracion. Este grafo es la estructura de datos central que determina que recursos pueden procesarse en paralelo y cuales deben esperar a que otros completen su operacion. La eficiencia de un despliegue de Terraform depende directamente de la calidad del grafo: un grafo con paralelismo maximo reduce significativamente el tiempo total de ejecucion, mientras que un grafo sobredeterminado con dependencias innecesarias lo extiende artificialmente.

Un grafo aciclico dirigido es una estructura donde los nodos representan recursos y los arcos representan relaciones de dependencia. La propiedad "aciclico" garantiza que no existan circulos: si el recurso A depende de B y B depende de C, entonces C no puede depender de A, lo cual eliminaria cualquier ambiguedad en el orden de ejecucion. La propiedad "dirigido" indica que las dependencias tienen una direccion: el arco apunta desde el recurso que depende hacia el recurso del cual depende. Terraform utiliza este grafo para calcular un orden de ejecucion topologico donde cada recurso se procesa solo despues de haber completado todos los recursos de los que depende.

## Construccion del grafo

Durante la fase de planificacion, Terraform analiza todos los bloques de configuracion y construye el grafo de forma iterativa. Primero registra todos los recursos como nodos del grafo, luego examina cada recurso en busca de referencias a otros recursos para crear los arcos correspondientes. Posteriormente incorpora las dependencias explicitas declaradas con `depends_on` y finalmente aplica restricciones de proveedor y de estado que puedan afectar el orden. El resultado es un grafo completo que refleja todas las relaciones de dependencia conocidas.

```hcl
resource "aws_vpc" "principal" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "publica_a" {
  vpc_id     = aws_vpc.principal.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "a"
}

resource "aws_subnet" "publica_b" {
  vpc_id     = aws_vpc.principal.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "b"
}

resource "aws_security_group" "web" {
  vpc_id = aws_vpc.principal.id
}

resource "aws_instance" "web_a" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.publica_a.id
  vpc_security_group_ids = [aws_security_group.web.id]
}

resource "aws_instance" "web_b" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.publica_b.id
  vpc_security_group_ids = [aws_security_group.web.id]
}
```

El grafo resultante tiene la siguiente estructura de niveles:

```
Nivel 0: aws_vpc.principal
Nivel 1: aws_subnet.publica_a, aws_subnet.publica_b, aws_security_group.web
Nivel 2: aws_instance.web_a, aws_instance.web_b
```

Terraform ejecuta todos los recursos de un nivel en paralelo y solo avanza al siguiente nivel cuando todos los recursos del actual han completado. Las dos subredes y el grupo de seguridad se crean simultaneamente en el nivel 1, y las dos instancias se crean en paralelo en el nivel 2 una vez que las subredes y el grupo de seguridad estan listos.

## Ejecucion paralela en la practica

Cuando se ejecuta `terraform apply`, la salida del plan muestra los recursos agrupados por el numero de operaciones paralelas que Terraform ejecutara. Cada linea indica el tipo de operacion (create, update, destroy) y el recurso afectado. Los recursos que no tienen dependencias entre si se marcan con la misma cantidad de signos `+` o `~` indicando que se ejecutaran simultaneamente.

```
Terraform will perform the following actions:

  # aws_vpc.principal will be created
  + resource "aws_vpc" "principal" {
      + cidr_block = "10.0.0.0/16"
    }

  # aws_subnet.publica_a will be created
  + resource "aws_subnet" "publica_a" {
      + vpc_id     = (known after apply)
      + cidr_block = "10.0.1.0/24"
    }

  # aws_subnet.publica_b will be created
  + resource "aws_subnet" "publica_b" {
      + vpc_id     = (known after apply)
      + cidr_block = "10.0.2.0/24"
    }

Plan: 4 to add, 0 to change, 0 to destroy.
```

En la salida real de un apply con recursos paralelos, Terraform muestra multiples operaciones ejecutandose simultaneamente con un indicador visual de progreso. El numero de operaciones paralelas por defecto es de 10, pero se puede ajustar con el argumento `-parallelism=n` en la linea de comandos.

> El paralelismo por defecto de 10 es un equilibrio entre velocidad y consumo de recursos locales. En entornos con muchos recursos o con proveedores cloud que tienen cuotas de tasa de solicitud, reducir el paralelismo puede evitar errores de throttling.

## Optimizacion del grafo

La forma en que se escribe la configuracion de Terraform afecta directamente la estructura del grafo y por tanto el rendimiento del despliegue. Cuando se utilizan referencias directas entre recursos, Terraform puede construir un grafo preciso con el maximo paralelismo posible. Sin embargo, el uso excesivo de `depends_on` o la creacion de cadenas de dependencias innecesarias puede crear cuellos de botella en el grafo.

```hcl
resource "aws_vpc" "principal" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "publica" {
  vpc_id = aws_vpc.principal.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id
}

resource "aws_route_table_association" "publica" {
  subnet_id      = aws_subnet.publica.id
  route_table_id = aws_route_table.publica.id
}
```

En este ejemplo, `aws_subnet.publica` y `aws_route_table.publica` no dependen una de otra, solo dependen de `aws_vpc.principal`. Terraform las ejecuta en paralelo. Luego `aws_route_table_association.publica` depende de ambas y se ejecuta despues. Si se hubiera agregado artificialmente una dependencia de `aws_route_table.publica` hacia `aws_subnet.publica` mediante `depends_on`, se habria eliminado el paralelismo del nivel 1 y las operaciones totales habrian aumentado.

Para inspeccionar el grafo resultante, Terraform dispone del comando `terraform graph` que genera una representacion visual del DAG en formato DOT. Esta representacion puede visualizarse con herramientas como Graphviz para identificar cuellos de botella o dependencias inesperadas.

```bash
terraform graph | dot -Tsvg > grafo.svg
```

El comando genera un archivo SVG con la representacion visual del grafo de dependencias, donde cada nodo es un recurso o modulo y cada arco es una relacion de dependencia. Los arcos con borde punteado indican dependencias implicitas, mientras que los solidos indican dependencias de modulo o de fuente de datos.

