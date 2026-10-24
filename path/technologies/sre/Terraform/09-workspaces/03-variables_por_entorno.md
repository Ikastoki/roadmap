# Variables por Entorno

La parametrizacion de configuraciones especificas de cada entorno es el complemento necesario de cualquier estrategia multi-entorno. Sin un mecanismo para asignar valores distintos a las mismas variables segun el entorno objetivo, el codigo de infraestructura seria rigido y obligaria a modificar archivos manualmente antes de cada despliegue. Terraform proporciona varios mecanismos para inyectar valores variables en tiempo de ejecucion, y la eleccion entre ellos determina la flexibilidad y mantenibilidad del proyecto.

## Archivos tfvars

Terraform permite definir valores para las variables de entrada mediante archivos con extension `.tfvars`. Estos archivos contienen asignaciones en sintaxis HCL y se cargan automaticamente o de forma explicita durante la ejecucion de `terraform apply` o `terraform plan`.

El archivo `terraform.tfvars` (o `terraform.tfvars.json`) tiene un comportamiento especial: Terraform lo carga automaticamente si existe en el directorio de trabajo, sin necesidad de indicarlo en la linea de comandos. Es el lugar natural para valores por defecto especificos del entorno o para configuraciones compartidas entre todos los miembros del equipo.

```hcl
# terraform.tfvars
region     = "eu-west-1"
proyecto   = "mi-aplicacion"
ambiente   = "dev"
```

Para entornos adicionales, se crean archivos con nombres descriptivos como `dev.tfvars`, `staging.tfvars` o `prod.tfvars`. Estos no se cargan automaticamente; deben pasarse explicitamente con el flag `-var-file`:

```bash
# Despliegue en produccion
terraform apply -var-file="prod.tfvars"

# Despliegue en staging
terraform apply -var-file="staging.tfvars"
```

Es posible combinar multiples archivos `-var-file` en una misma invocacion. En caso de conflicto entre archivos, el ultimo especificado tiene prioridad. El orden de precedencia completo de Terraform para la resolucion de variables es:

1. Variables de entorno (`TF_VAR_<nombre>`)
2. `terraform.tfvars` o `terraform.tfvars.json`
3. Archivos `*.auto.tfvars` o `*.auto.tfvars.json` (en orden alfabetico)
4. Archivos pasados con `-var-file` o `-var`

Este orden implica que `-var-file` puede sobrescribir valores definidos en `terraform.tfvars`, pero las variables de entorno a su vez pueden sobrescribir a `-var-file`. En la practica, los archivos tfvars son el mecanismo preferido porque se versionan junto al codigo y proporcionan un registro explicito de los valores utilizados en cada despliegue.

## Estructura tipica con tfvars por entorno

Combinado con la estrategia de directorios separados, el patron mas comun es almacenar un archivo tfvars por entorno dentro del directorio correspondiente:

```
environments/
  dev/
    main.tf
    variables.tf
    backend.tf
    dev.tfvars           # Valores especificos de desarrollo
  staging/
    main.tf
    variables.tf
    backend.tf
    staging.tfvars       # Valores especificos de staging
  prod/
    main.tf
    variables.tf
    backend.tf
    prod.tfvars          # Valores especificos de produccion
```

El contenido de `prod.tfvars` podria ser:

```hcl
instance_type      = "t3.large"
min_size           = 3
max_size           = 10
ambiente           = "prod"
enable_monitoring  = true
```

Mientras que `dev.tfvars` probablemente usaria valores minimos:

```hcl
instance_type      = "t2.micro"
min_size           = 1
max_size           = 2
ambiente           = "dev"
enable_monitoring  = false
```

La separacion en archivos mantiene el codigo de los modulos limpio y generico, mientras que los valores concretos de cada entorno quedan aislados en archivos independientes. Un cambio en el tamao de las instancias de produccion no require tocar el codigo de los modulos ni el de otros entornos.

## Condicionales basadas en workspace

Cuando se utiliza la estrategia de workspaces en lugar de directorios separados, no existen archivos tfvars distintos para cada entorno, ya que todo comparte el mismo directorio. En este escenario, la variable `terraform.workspace` puede utilizarse para aplicar logica condicional que seleccione valores segun el entorno activo.

```hcl
variable "instance_type" {
  description = "Tipo de instancia segun entorno"
  type        = string
}

locals {
  instance_type_map = {
    dev    = "t2.micro"
    staging = "t3.small"
    prod   = "t3.large"
  }
  ambiente = terraform.workspace
}

resource "aws_instance" "servidor" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = lookup(local.instance_type_map, local.ambiente, "t2.micro")

  tags = {
    Name = "servidor-${local.ambiente}"
  }
}
```

La funcion `lookup` devuelve el valor asociado a la clave `local.ambiente` dentro del mapa. Si el workspace no existe como clave (por ejemplo, un workspace llamado `testing` que no esta en el mapa), se utiliza el valor por defecto `"t2.micro"`. Este patron evita largas cadenas de condicionales `if-else` y mantiene la logica centralizada en un unico mapa.

Una alternativa mas explicita utiliza la funcion `terraform.workspace` directamente en expresiones condicionales:

```hcl
instance_type = terraform.workspace == "prod" ? "t3.large" : "t2.micro"
```

Aunque esta sintaxis es mas compacta, se vuelve ilegible cuando intervienen mas de dos entornos o cuando las condiciones se anidan. El mapa con `lookup` es casi siempre preferible por su escalabilidad y facilidad de lectura.

> El uso excesivo de condicionales basadas en `terraform.workspace` es un sintoma de que la estrategia de workspaces se esta quedando pequena. Cuando los condicionales se multiplican o la logica de seleccion supera unas pocas lineas, es senal de que conviene migrar a directorios separados con archivos tfvars independientes.

## Archivos auto.tfvars

Terraform tambien reconoce archivos con el patron `*.auto.tfvars` y `*.auto.tfvars.json`, que se cargan automaticamente sin necesidad de `-var-file`. La diferencia con `terraform.tfvars` es que pueden existir multiples archivos `auto.tfvars` y se procesan en orden alfabetico.

Este mecanismo resulta util cuando se quiere segregar las variables por categoria sin renunciar a la carga automatica:

```
network.auto.tfvars       # Variables de red
compute.auto.tfvars       # Variables de computo
monitoring.auto.tfvars    # Variables de monitorizacion
```

En un pipeline de CI/CD, los archivos `auto.tfvars` pueden generarse dinamicamente durante la ejecucion, combinando valores predeterminados del repositorio con secretos inyectados por el sistema de integracion continua.

## Precedencia practica en entornos reales

En la mayoria de proyectos que usan directorios separados, la configuracion de variables sigue este esquema:

- `terraform.tfvars`: valores compartidos por todos los miembros del equipo (region por defecto, prefijo de nombres, etc.).
- `prod.tfvars`, `dev.tfvars`, etc.: valores especificos de cada entorno.
- `TF_VAR_*`: secretos y tokens que no deben estar en el repositorio.
- `-var`: valores puntuales para pruebas rapidas o sobreescrituras temporales.

Este sistema de capas proporciona flexibilidad sin sacrificar claridad: los valores estables y compartidos viven en archivos versionados, los secretos se inyectan desde el entorno de ejecucion y las sobreescrituras ad-hoc quedan en la linea de comandos, visibles en los logs del pipeline.

