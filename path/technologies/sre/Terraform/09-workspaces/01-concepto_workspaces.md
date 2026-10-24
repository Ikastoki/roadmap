# Concepto de Workspaces

Cuando se gestiona infraestructura con Terraform, cada conjunto de configuraciones produce un archivo de estado (`terraform.tfstate`) que mapea los recursos declarados en codigo contra los recursos reales en el proveedor cloud. Por defecto, existe una relacion uno a uno entre el directorio de trabajo y el archivo de estado. Los workspaces rompen esta relacion unica al permitir que un mismo directorio y un mismo conjunto de archivos de configuracion gestionen multiples archivos de estado independientes, cada uno asociado a un workspace diferente.

Un workspace no es mas que un identificador con nombre que Terraform utiliza para discriminar que archivo de estado debe leer y escribir en cada operacion. Si el backend es local, los estados se almacenan en `terraform.tfstate.d/<nombre_workspace>/`; si el backend es remoto (S3, Terraform Cloud, etc.), cada workspace se corresponde con una clave o ruta distinta dentro del mismo backend. Esto permite ejecutar `terraform plan` y `terraform apply` contra entornos logicamente separados (desarrollo, puesta en escena, produccion) sin duplicar ni modificar el codigo fuente.

## Gestion basica de workspaces

Terraform proporciona un subcomando `workspace` con varias operaciones. Al inicializar un proyecto, Terraform crea automaticamente un workspace llamado `default`. La mayoria de proyectos en solitario o de un solo entorno nunca necesitan salir de este workspace, pero en cuanto aparecen multiples entornos, la gestion de workspaces se vuelve indispensable.

El listado de workspaces se obtiene con `terraform workspace list`. El workspace activo se marca con un asterisco:

```
$ terraform workspace list
  default
* dev
  staging
  prod
```

Para crear un nuevo workspace se usa `terraform workspace new <nombre>`. Al crearlo, Terraform cambia automaticamente al nuevo workspace. El estado inicial esta vacio: no conoce ningun recurso, por lo que el primer `apply` creara toda la infraestructura desde cero.

```
$ terraform workspace new dev
Created and switched to workspace "dev"!
```

Para cambiar entre workspaces existentes se utiliza `terraform workspace select <nombre>`:

```
$ terraform workspace select prod
Switched to workspace "prod".
```

La eliminacion de un workspace requiere usar `terraform workspace delete <nombre>`. Terraform solo permite eliminar workspaces cuyo estado este vacio (sin recursos gestionados). Si el workspace aun contiene recursos, es necesario destruirlos antes con `terraform destroy` o, si se desea preservar la infraestructura pero dejar de gestionarla, usar `terraform state rm` para cada recurso.

> Los nombres de workspace tienen restricciones: deben ser caracteres alfanumericos, guiones y guiones bajos, y no pueden exceder los 128 caracteres. El workspace `default` no se puede eliminar.

## La variable terraform.workspace

Terraform expone una variable predefinida de solo lectura llamada `terraform.workspace` que contiene el nombre del workspace activo en tiempo de ejecucion. Esta variable es la pieza central que conecta la logica del codigo con el entorno concreto que se esta desplegando.

El uso mas inmediato es etiquetar recursos con el nombre del workspace para identificarlos visualmente en la consola del proveedor cloud:

```hcl
resource "aws_instance" "servidor" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name        = "servidor-${terraform.workspace}"
    Environment = terraform.workspace
  }
}
```

En este ejemplo, si el workspace activo es `dev`, la instancia se creara con el tag `Name = "servidor-dev"`; si es `prod`, sera `servidor-prod`. Esto evita colisiones de nombres cuando multiples entornos comparten la misma cuenta cloud y proporciona trazabilidad inmediata sobre a que entorno pertenece cada recurso.

Sin embargo, `terraform.workspace` no debe utilizarse para seleccionar valores de configuracion que diffieran entre entornos (como el tamao de una instancia o el numero de replicas). Para ese proposito existen mecanismos mas adecuados como los archivos `tfvars` o las variables de entorno, que se exploran en los capitulos siguientes. Usar `terraform.workspace` para logica condicional compleja acopla el codigo a los nombres de los workspaces, lo que dificulta su reutilizacion y prueba.

## Implicaciones del diseno

Cada workspace mantiene su propio archivo de estado independiente. Esto significa que dos workspaces pueden contener recursos con nombres identicos sin conflicto, porque Terraform los considera recursos distintos al estar registrados en estados separados. Tambien implica que las operaciones de bloqueo (state locking) son independientes por workspace: se puede estar aplicando cambios en `staging` mientras otro miembro del equipo aplica en `prod` sin interferencias.

El aislamiento, no obstante, no es completo. Todos los workspaces comparten el mismo backend y las mismas credenciales del proveedor cloud. Si el backend es un bucket S3, todos los workspaces escriben en el mismo bucket, aunque con claves diferentes. Si las credenciales de AWS permiten crear o destruir recursos en produccion, un error en la seleccion del workspace puede tener consecuencias graves. Por esta razon, muchas organizaciones combinan workspaces con pipelines de CI/CD que evitan la interaccion manual y anaden capas de aprobacion antes de afectar entornos critico.

