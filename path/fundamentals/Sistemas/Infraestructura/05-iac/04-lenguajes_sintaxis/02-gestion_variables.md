# Gestion de variables

La separacion entre la definicion de infraestructura y los valores concretos que la parametrizan es uno de los principios fundamentales del IaC. Sin variables, cada entorno (desarrollo, staging, produccion) requeriria una copia completa de los manifiestos, lo que multiplica el riesgo de inconsistencias y dificulta la mantenibilidad. Las variables introducen un nivel de abstraccion que permite reutilizar la misma logica de despliegue con valores distintos, manteniendo la fuente unica de verdad.

Cada herramienta de IaC aborda la gestion de variables con su propio modelo. Terraform distingue entre declaracion de variables (tipo, descripcion, valor por defecto, validacion), asignacion de valores (archivos `.tfvars`, argumentos de linea de comandos, variables de entorno) y consumo en recursos. Pulumi utiliza un sistema de configuracion basado en archivos YAML (`Pulumi.<stack>.yaml`) y un API tipada que genera objetos con las propiedades declaradas. CloudFormation emplea un bloque `Parameters` con un esquema estricto de tipos y restricciones validadas por la propia plataforma de AWS.

## Declaracion de tipos

La tipificacion de variables no es un capricho de sintaxis: es la primera linea de defensa contra errores de configuracion que, en entornos de produccion, pueden tener consecuencias economicas o de seguridad. Un identificador de subred que se recibe como cadena vacia, un numero de puerto fuera de rango o un booleano confundido con una cadena son fallos que un sistema de tipos puede detectar antes de que se ejecute ningun plan de despliegue.

**Terraform: tipos primitivos y coleccion**

```hcl
variable "nombre_proyecto" {
  type        = string
  description = "Nombre del proyecto, usado como prefijo en todos los recursos"
  validation {
    condition     = length(var.nombre_proyecto) > 3 && length(var.nombre_proyecto) <= 24
    error_message = "El nombre debe tener entre 4 y 24 caracteres."
  }
}

variable "num_instancias" {
  type    = number
  default = 2
  validation {
    condition     = var.num_instancias >= 1 && var.num_instancias <= 10
    error_message = "El numero de instancias debe estar entre 1 y 10."
  }
}

variable "habilitar_ssl" {
  type    = bool
  default = true
}

variable "subnets_privadas" {
  type = list(string)
}

variable "configuracion_tags" {
  type = map(string)
  default = {
    ManagedBy = "terraform"
    Ambiente  = "dev"
  }
}

variable "regla_firewall" {
  type = object({
    puerto     = number
    protocolo  = string
    cidr_origen = list(string)
  })
}
```

El bloque `validation` en Terraform permite definir expresiones condicionales que deben evaluarse a `true` para que la operacion continúe. Estas validaciones se ejecutan durante `terraform plan` y `terraform apply`, no en el momento de la declaracion, lo que significa que las dependencias entre variables deben existir en tiempo de ejecucion.

**Pulumi (TypeScript): tipos nativos del lenguaje**

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();

const nombreProyecto = config.require("nombre-proyecto");
const numInstancias = config.getNumber("num-instancias") ?? 2;
const habilitarSsl = config.getBoolean("habilitar-ssl") ?? true;
const subnetsPrivadas = config.getObject<string[]>("subnets-privadas") ?? [];
```

En Pulumi, la tipificacion se aprovecha directamente del lenguaje host. TypeScript proporciona tipos en tiempo de compilacion, lo que detecta errores antes de que se ejecute ningun comando. El metodo `config.require()` lanza una excepcion si la clave no existe, lo que obliga a definir todas las variables necesarias antes del despliegue.

**CloudFormation: parametros con restricciones**

```yaml
Parameters:
  NombreProyecto:
    Type: String
    Description: Nombre del proyecto (prefijo de recursos)
    MinLength: 4
    MaxLength: 24
    AllowedPattern: "[a-z0-9-]+"
    ConstraintDescription: "Solo minusculas, numeros y guiones"

  NumInstancias:
    Type: Number
    Default: 2
    MinValue: 1
    MaxValue: 10

  HabilitarSsl:
    Type: String
    AllowedValues:
      - "true"
      - "false"
    Default: "true"
```

> CloudFormation no soporta un tipo booleano nativo. Los valores booleanos se representan como cadenas `"true"` o `"false"`, lo cual requiere conversions explícitas en las expresiones del template. Esta es una de las razones por las que CloudFormation se percibe como menos ergonomica que herramientas con soporte de tipos nativos.

## Metodos de inyeccion de valores

Una vez declaradas las variables, el siguiente problema es como proporcionarles valores en cada ejecucion. Los metodos disponibles varian en cuanto a seguridad, comodidad y nivel de prioridad.

**Archivos de variables (.tfvars)**

```hcl
# produccion.tfvars
nombre_proyecto = "miapp"
num_instancias  = 5
habilitar_ssl   = true

configuracion_tags = {
  Environment = "production"
  Team        = "platform"
  CostCenter  = "infra-001"
}
```

```bash
terraform apply -var-file="produccion.tfvars"
```

Los archivos `.tfvars` permiten parametrizar ejecuciones completas sin exponer valores sensibles en el historial de comandos. Es recomendable versionar archivos de valores para entornos no sensibles y mantener los valores de produccion fuera del repositorio, en un gestor de secretos o en una plataforma de CI/CD.

**Argumentos de linea de comandos**

```bash
terraform apply -var="nombre_proyecto=miapp" -var="num_instancias=3"
pulumi config set num-instancias 3
```

Los argumentos CLI son utiles para pruebas rapidas, pero inseguros en produccion: quedan registrados en el historial del shell y en los logs de los sistemas de ejecucion.

**Variables de entorno del sistema**

```bash
export TF_VAR_nombre_proyecto="miapp"
export TF_VAR_num_instancias=3
terraform apply
```

Terraform busca variables con el prefijo `TF_VAR_` y el nombre exacto de la variable declarada. Este metodo es util en entornos CI/CD donde las variables de entorno se inyectan de forma segura desde el gestor de secretos del pipeline.

## Reglas de resolucion de precedencia

Cuando una variable puede obtener su valor de multiples fuentes, Terraform aplica una cadena de precedencia estricta. La fuente con mayor prioridad siempre anula a las anteriores:

1. Variables de entorno (`TF_VAR_nombre`)
2. Argumentos de linea de comandos (`-var "nombre=valor"`)
3. Archivos especificados (`-var-file="archivo.tfvars"`)
4. Archivo `terraform.tfvars` o `*.auto.tfvars` en el directorio de ejecucion
5. Valor por defecto definido en el bloque `variable`

> En Pulumi, la precedencia se resuelve de forma similar: los valores establecidos con `pulumi config set` tienen prioridad sobre los valores por defecto del programa. Si se utiliza un archivo de configuracion en el repositorio, los valores de la pila remota (gestionados desde la consola de Pulumi) tienen mayor prioridad que los locales.

