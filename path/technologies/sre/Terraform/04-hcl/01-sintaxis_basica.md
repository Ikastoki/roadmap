# Sintaxis basica de HCL

HCL (HashiCorp Configuration Language) es el lenguaje declarativo que utiliza Terraform para definir infraestructura como codigo. A diferencia de lenguajes de proposito general como Python o Go, HCL esta disenado especificamente para describir recursos, sus atributos y las relaciones entre ellos. Su sintaxis combina elementos de JSON con una legibilidad cercana al lenguaje natural, eliminando la necesidad de comillas innecesarias y proporcionando una estructura jerarquica clara mediante bloques anidados.

El diseno de HCL prioriza la legibilidad humana sobre la facilidad de parseo automatico. Aunque Terraform tambien acepta configuraciones en JSON puro, la mayoria de los proyectos utilizan la sintaxis nativa de HCL por su concision y expresividad.

## Bloques

Los bloques son la unidad fundamental de configuracion en HCL. Definen recursos, proveedores, variables, outputs y cualquier otro elemento de infraestructura. La estructura basica de un bloque sigue el patron:

```hcl
tipo "nombre" "etiqueta" {
  # contenido del bloque
}
```

El `tipo` indica que clase de elemento se define (resource, data, provider, variable, output, etc.). El `nombre` es un identificador unico dentro del tipo. La `etiqueta` es opcional y depende del tipo de bloque: los recursos y data sources requieren una etiqueta adicional que actua como identificador dentro del bloque.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

provider "aws" {
  region = "us-east-1"
}

variable "ambiente" {
  type    = string
  default = "produccion"
}
```

El bloque `resource` es el mas utilizado. Requiere dos etiquetas: el tipo de recurso (`aws_instance`) y el nombre local (`web`). La combinacion `resource "aws_instance" "web"` debe ser unica en todo el proyecto.

> En el sistema de tipos de Terraform, los bloques no son valores asignables a variables. Solo pueden definirse en la raiz del archivo de configuracion. Sin embargo, el contenido de los bloques (sus atributos) sigue las reglas de tipos de HCL.

## Atributos

Los atributos son pares `clave = valor` que configuran las propiedades de un bloque. La sintaxis es siempre `nombre = expresion`, donde `nombre` es un identificador y `expresion` puede ser un valor literal, una referencia a otro recurso, una llamada a funcion o una expresion condicional.

```hcl
resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t2.micro"
  disable_api_termination = true

  tags = {
    Name        = "Servidor Web"
    Environment = var.ambiente
  }
}
```

Los atributos pueden referenciar valores definidos en otros bloques mediante la sintaxis `<tipo>.<nombre>.<atributo>`. Por ejemplo, `aws_vpc.main.id` accede al atributo `id` del recurso `aws_vpc` llamado `main`. Esta capacidad de referencia es lo que permite a Terraform construir el grafo de dependencias implicitamente.

## Tipos de datos

HCL soporta seis tipos de datos fundamentales que se pueden combinar para construir estructuras mas complejas:

**Tipos primitivos:**

| Tipo | Descripcion | Ejemplo |
|------|-------------|---------|
| `string` | Cadena de texto | `"ami-0c55b159cbfafe1f0"` |
| `number` | Numero entero o decimal | `42`, `3.14` |
| `bool` | Booleano | `true`, `false` |

**Tipos complejos:**

| Tipo | Descripcion | Ejemplo |
|------|-------------|---------|
| `list` | Lista ordenada de elementos del mismo tipo | `["us-east-1", "us-west-2"]` |
| `map` | Coleccion clave-valor con claves de tipo string | `{ Name = "web", Env = "prod" }` |
| `set` | Conjunto no ordenado de elementos unicos del mismo tipo | `["a", "b", "c"]` |

**Tipos estructurales:**

| Tipo | Descripcion | Ejemplo |
|------|-------------|---------|
| `object` | Conjunto de atributos con tipos especificos | `{ name = "web", port = 80, ssl = true }` |
| `tuple` | Secuencia de elementos de tipos potencialmente distintos | `["web", 80, true]` |

La diferencia fundamental entre `list` y `tuple` es que `list` exige que todos los elementos sean del mismo tipo, mientras que `tuple` permite tipos mixtos. Analogamente, `map` exige que todos los valores sean del mismo tipo, mientras que `object` permite definirlos individualmente.

```hcl
# Lista homogenea
variable "regiones" {
  type    = list(string)
  default = ["us-east-1", "eu-west-1"]
}

# Objeto con tipos mixtos
variable "instancia" {
  type = object({
    name    = string
    port    = number
    ssl     = bool
    tags    = map(string)
  })
  default = {
    name = "web"
    port = 443
    ssl  = true
    tags = { Environment = "prod" }
  }
}

# Tupla con tipos mixtos
variable "config_db" {
  type    = tuple([string, number, bool])
  default = ["postgresql", 5432, true]
}
```

HCL realiza coercion automatica de tipos en ciertos casos. Un `number` puede convertirse a `string` cuando se usa en contexto de texto, y viceversa. Sin embargo, las conversiones entre tipos complejos requieren funciones explicitas como `tostring()`, `tonumber()`, `tolist()`, `tomap()`.

## Comentarios

HCL soporta tres estilos de comentario:

```hcl
# Comentario de linea con almohadilla

// Comentario de linea con doble barra

/* Comentario
   de bloque
   multilinea */
```

El estilo con `#` es el mas utilizado en la comunidad Terraform por tradicion del lenguaje. Los comentarios de bloque `/* */` son utiles para comentar secciones enteras de configuracion durante el desarrollo o la depuracion.

```hcl
# Configuracion del proveedor AWS
provider "aws" {
  region = "us-east-1"
}

/* Descomentar para habilitar la region secundaria
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
*/
```

> Los comentarios no se incluyen en el estado de Terraform ni afectan al comportamiento de la infraestructura. Son exclusivamente para el desarrollador.

## Expresiones e interpolacion

HCL permite incrustar expresiones dentro de strings mediante la sintaxis de interpolacion `${ ... }`. El uso mas comun es referenciar atributos de otros recursos:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "Servidor-${var.ambiente}"
  }
}

output "public_ip" {
  value = "La IP publica es ${aws_instance.web.public_ip}"
}
```

Las expresiones dentro de `${}` pueden incluir operadores aritmeticos, comparaciones logicas, llamadas a funciones y operaciones con colecciones. Terraform evalua estas expresiones durante el plan, no en tiempo de ejecucion.

```hcl
# Operadores aritmeticos
count = var.replicas * 2

# Operadores logicos
instance_type = var.env == "prod" ? "t3.large" : "t2.micro"

# Llamadas a funciones
name = "servidor-${lower(var.ambiente)}"

# Operaciones con listas
subnets = slice(data.aws_subnet_ids.all.ids, 0, 2)
```

