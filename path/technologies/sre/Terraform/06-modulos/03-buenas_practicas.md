# Buenas Practicas en Modulos

Tener la capacidad de crear modulos no garantiza que estos sean utiles, mantenibles o seguros. Un modulo mal disenado puede ser peor que la duplicacion de codigo: introduce una capa de abstraccion que oculta complejidad no deseada, fuerza a sus consumidores a lidiar con variables mal definidas y genera dependencias fragiles entre equipos. Las buenas practicas en torno a modulos no son reglas arbitrarias, sino lecciones extraidas de proyectos que han escalado la gestion de infraestructura con Terraform durante anos.

## Principio de responsabilidad unica

Cada modulo deberia encapsular una unidad logica de infraestructura y solo una. Si un modulo crea una VPC, configura un balanceador y despliega una base de datos, esta violando el principio de responsabilidad unica. Las consecuencias son inmediatas: el modulo se vuelve dificil de reutilizar (no todo el mundo necesita una base de datos junto a su VPC), sus opciones de configuracion se multiplican (variables condicionales para activar o desactivar partes) y el riesgo de efectos colaterales aumenta.

```hcl
# Mal: modulo que hace demasiadas cosas
module "infraestructura_completa" {
  source = "./modules/todo_en_uno"

  crear_vpc    = true
  crear_rds    = true
  crear_alb    = true
  entorno      = "prod"
  cidr         = "10.0.0.0/16"
  db_password  = var.db_password
}
```

Un modulo con multiples responsabilidades suele terminar con decenas de variables, muchas de ellas condicionales, y un `main.tf` salpicado de `count` y `for_each` para decidir que recursos crear. Esto convierte el modulo en un monolito configurable que ningun equipo quiere mantener.

```hcl
# Bien: modulos atomicos y composibles
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "rds" {
  source     = "./modules/rds"
  vpc_id     = module.vpc.vpc_id
  subnets    = module.vpc.subredes_privadas
  password   = var.db_password
}

module "alb" {
  source  = "./modules/alb"
  vpc_id  = module.vpc.vpc_id
  subnets = module.vpc.subredes_publicas
}
```

Cada modulo hace una cosa y la hace bien. La composicion --el arte de combinar modulos pequenos para construir sistemas complejos-- ocurre en el modulo root o en un modulo orquestador de alto nivel.

## Documentar variables y outputs

Un modulo sin documentacion es inutilizable por cualquier persona que no sea su autor. La documentacion no es un lujo; es parte integral del contrato publico del modulo. Cada variable debe incluir `description` y `type`. Cada output debe explicar que valor devuelve y en que formato.

```hcl
variable "puerto_aplicacion" {
  description = "Puerto TCP donde escuchara la aplicacion dentro del contenedor"
  type        = number
  default     = 8080

  validation {
    condition     = var.puerto_aplicacion > 0 && var.puerto_aplicacion < 65536
    error_message = "El puerto debe estar entre 1 y 65535."
  }
}
```

La inclusion de bloques `validation` en las variables anade una capa extra de seguridad: el modulo rechaza entradas invalidas en tiempo de planificacion, antes de que los errores se manifiesten como fallos en la nube.

Para outputs, la documentacion debe indicar no solo el valor sino su contexto. Un output que devuelve un ARN no es util si el consumidor no sabe que recurso exactamente esta referenciando.

```hcl
output "alb_arn" {
  description = "ARN del Application Load Balancer creado para el servicio web"
  value       = aws_lb.this.arn
}
```

Ademas de la documentacion en el codigo, el `README.md` del modulo debe incluir un ejemplo de uso completo, las variables requeridas y opcionales, los outputs principales y cualquier consideracion especial (como prerequisitos de proveedores o dependencias externas). Herramientas como `terraform-docs` pueden generar automaticamente el README a partir de los comentarios del codigo, garantizando que documentacion y codigo no diverjan.

## Directorio examples/

Cada modulo deberia incluir un directorio `examples/` con configuraciones funcionales que demuestren como se utiliza. Estos ejemplos no son meras ilustraciones: deben ser ejecutables con `terraform apply` (con las credenciales adecuadas) y servir como prueba de que el modulo funciona.

```
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── README.md
└── examples/
    └── basico/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

El ejemplo basico muestra el caso de uso mas simple. Si el modulo soporta distintos modos operativos, cada modo deberia tener su propio ejemplo.

```hcl
# examples/basico/main.tf
module "vpc_ejemplo" {
  source = "../../"

  nombre      = "ejemplo"
  cidr_bloque = "10.0.0.0/16"
  subredes_cidr = [
    "10.0.1.0/24",
    "10.0.2.0/24",
  ]
  zonas = ["us-east-1a", "us-east-1b"]
}

output "vpc_id" {
  value = module.vpc_ejemplo.vpc_id
}
```

> La referencia `source = "../../"` en el ejemplo apunta al directorio raiz del modulo, no a una copia. Esto garantiza que el ejemplo siempre prueba la version actual del modulo. Para pruebas mas rigurosas se pueden utilizar herramientas como Terratest para ejecutar los ejemplos en un entorno real y verificar el resultado.

Los ejemplos tambien funcionan como tests de integracion: si un cambio rompe el ejemplo, el modulo esta roto. Integrar la ejecucion de los ejemplos en un pipeline de CI/CD proporciona verificacion automatica de que el modulo sigue siendo funcional despues de cada modificacion.

## Composicion de modulos

La composicion es la estrategia opuesta al monolito. En lugar de construir modulos grandes que intentan cubrir todos los casos, se construyen modulos atomicos que se ensamblan en configuraciones de mayor nivel. La composicion no ocurre dentro del modulo, sino fuera, en el modulo root o en modulos orquestadores.

```hcl
# modulo orquestador: modules/infraestructura_web/
module "vpc" {
  source = "../vpc"
  cidr   = var.cidr
}

module "alb" {
  source     = "../alb"
  vpc_id     = module.vpc.vpc_id
  subnets    = module.vpc.subredes_publicas
  certificado_arn = var.certificado_arn
}

module "ecs" {
  source        = "../ecs"
  vpc_id        = module.vpc.vpc_id
  subnets       = module.vpc.subredes_privadas
  alb_arn       = module.alb.arn
  puerto        = var.puerto_app
  imagen_docker = var.imagen
}
```

Este modulo orquestador "infraestructura_web" no anade nuevos recursos; se limita a conectar modulos existentes, pasando outputs de unos como inputs de otros. El valor anadido reside en la configuracion de las conexiones y en la reduction de la carga cognitiva para quien despliega una aplicacion web tipica.

La composicion efectiva requiere que los modulos atomicos esten disenados para ser conectables: sus outputs deben exponer exactamente los valores que otros modulos necesitan, ni mas ni menos. Un modulo de VPC que devuelve los IDs de las subredes publicas y privadas permite que un modulo de ALB se conecte a las publicas y un modulo de RDS a las privadas. Si el modulo de VPC solo devolviera un lista unica de subredes sin distinguir su tipo, la composicion se volveria imposible sin modificar el modulo.

## Otras consideraciones

**Modulos planos, no anidados en exceso.** Un modulo que llama a otro modulo que llama a otro modulo crea una cadena de dependencias dificil de depurar. La profundidad recomendada es de dos niveles: root utiliza modulos hijos, y los hijos no deberian llamar a otros modulos a menos que haya una razon solida (por ejemplo, un modulo de Kubernetes que necesita crear nodos y usa un modulo de auto scaling).

**Evitar modulos que envuelven un unico recurso.** Crear un modulo para un solo recurso rara vez merece la pena. La excepcion es cuando ese recurso requiere una configuracion repetitiva con muchas opciones (como un bucket S3 con politicas, versionado, cifrado y replicacion) que se repite en multiples proyectos.

**No exponer variables que no sean necesarias.** Cada variable publica es un compromiso de mantenimiento. Una vez que alguien usa esa variable, cambiarla o eliminarla requiere coordinacion. Es mejor comenzar con una interfaz minima e ir ampliandola segun la demanda.

**Prefijar outputs para evitar colisiones.** Cuando multiples instancias del mismo modulo conviven en un mismo root, los outputs se referencian como `module.nombre.output`. Aun asi, es buena practica que los outputs tengan nombres descriptivos y no genericos como `id` o `arn`. Usar `vpc_id` en lugar de `id` elimina ambiguedad.

