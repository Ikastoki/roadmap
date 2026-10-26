# Dependencias implicitas

Cuando se declara infraestructura como codigo, el orden en que los recursos se crean, modifican o destruyen no es arbitrario: una subred no puede existir sin una VPC que la contenga, una instancia no puede unirse a un grupo de seguridad que aun no existe, y un registro DNS no puede apuntar a un balanceador de carga todavia no desplegado. Terraform resuelve este problema construyendo automaticamente un grafo de dependencias a partir de las referencias que existen entre los atributos de los recursos. Esta capacidad se denomina **dependencias implicitas** y constituye el mecanismo fundamental que permite al herramienta determinar el orden de ejecucion sin intervencion manual del operador.

El motor de Terraform analiza cada bloque de recurso y detecta cuando un atributo de un recurso depende del valor de otro recurso. En el momento en que escribes `vpc_id = aws_vpc.principal.id` dentro de un bloque de subred, Terraform comprende que la subred requiere que la VPC exista primero. Esta relacion se registra internamente como un arco dirigido en el grafo de dependencias y garantiza que la secuencia de operaciones sea siempre coherente, tanto durante la creacion inicial como en actualizaciones posteriores.

## Mecanismo de deteccion

Terraform examina las expresiones dentro de cada bloque de configuracion en busca de referencias a otros recursos, fuentes de datos o variables con valores derivados. Cuando encuentra una referencia como `aws_vpc.principal.id`, `aws_security_group.web.id` o `data.aws_ami.amazon_linux.id`, registra automaticamente una dependencia del recurso actual hacia el recurso referenciado. Este proceso ocurre durante la fase de planificacion, antes de que se ejecute cualquier accion real sobre la infraestructura.

```hcl
resource "aws_vpc" "principal" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "publica" {
  vpc_id     = aws_vpc.principal.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_subnet" "privada" {
  vpc_id     = aws_vpc.principal.id
  cidr_block = "10.0.2.0/24"
}
```

En este ejemplo, Terraform infiere que tanto `aws_subnet.publica` como `aws_subnet.privada` dependen de `aws_vpc.principal`. La VPC se creara siempre primero, y ambas subredes se crearan en paralelo despues, ya que no existen referencias cruzadas entre ellas. Si mas adelante se agrega un recurso que dependa de ambas subredes, el grafo se expande para incluir esa nueva relacion.

## Referencias indirectas y dependencias ocultas

No todas las dependencias son evidentes a simple vista. Cuando un modulo o un recurso utilice un atributo calculado a partir de la salida de otro, Terraform rastrea la cadena completa de referencias hasta encontrar el origen. Esto incluye expresiones condicionales, funciones de transformacion y composiciones de cadenas que involucren identificadores de otros recursos.

```hcl
resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
}

resource "aws_route_table_association" "publica_assoc" {
  subnet_id      = aws_subnet.publica.id
  route_table_id = aws_route_table.publica.id
}
```

Aqui existen multiples dependencias encadenadas: la ruta del gateway depende del `aws_internet_gateway.gw`, la asociacion de tabla de rutas depende tanto de la subred como de la tabla de rutas, y esta ultima depende de la VPC. Terraform traza todas estas relaciones y construye un grafo que refleja la totalidad de las interconexiones.

> Si un recurso A referencia `count = aws_vpc.principal.az`, Terraform tambien registra la dependencia aunque no sea una referencia de tipo ID. Cualquier uso de un atributo de otro recurso genera una dependencia implicita.

## Limitaciones de las dependencias implicitas

Las dependencias implicitas solo se detectan cuando existe una referencia directa entre atributos. Existen situaciones donde dos recursos estan logicamente relacionados pero no hay ninguna referencia textual entre ellos. Un ejemplo clasico es un recurso que necesita que un permiso IAM se propague antes de poder operar, o un servicio cloud que requiere un tiempo de propagacion despues de crear un dominio DNS antes de que los certificados SSL puedan validarse. En estos casos, la ausencia de una referencia directa significa que Terraform no creara la dependencia necesaria y podria intentar crear recursos en un orden incorrecto.

Otra limitacion ocurre cuando se utiliza `count` o `for_each` con expresiones complejas. Aunque Terraform maneja bien los casos tipicos, ciertos patrones de meta-argumentos pueden generar ambiguedades en el grafo que requieren intervencion explicita mediante `depends_on`.

```hcl
resource "aws_instance" "web" {
  count = var.num_instancias
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}

resource "aws_eip" "web" {
  count    = var.num_instancias
  instance = aws_instance.web[count.index].id
}
```

En este patron, la dependencia se establece correctamente porque `aws_eip` referencia directamente `aws_instance.web[count.index]`. Sin embargo, si la relacion fuera solo temporal o basada en convencion de nombres sin referenciar atributos, Terraform no la detectaria y necesitaria un `depends_on` explicito.

