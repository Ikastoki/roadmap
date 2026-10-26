# Estrategias de aislamiento en multiples entornos

Un despliegue de infraestructura real no opera sobre un unico entorno: el ciclo de vida de una aplicacion requiere al menos un entorno de desarrollo, uno de pruebas y otro de produccion. Cada uno de ellos consume recursos, posee configuraciones distintas y, fundamentalmente, debe permanecer aislado del resto para evitar que un cambio experimental provoque un incidente en produccion. La pregunta tecnica que surge entonces es como gestionar multiples entornos con la misma base de codigo de Terraform (o cualquier otra herramienta de IaC) sin generar conflictos de estado, colisiones de nombres o accesos no autorizados. Existen tres estrategias principales de aislamiento, cada una con un trade-off distinto entre simplicidad, flexibilidad y seguridad.

## Workspaces: multiples estados para un mismo codigo

Terraform workspaces permiten crear multiples estados independientes a partir del mismo conjunto de archivos `.tf`. Internamente, cada workspace almacena su propio `terraform.tfstate` en el backend configurado (local, S3, Consul, etc.), de modo que los recursos gestionados por un workspace no interfieren con los de otro. Esta mecanica es util para entornos identicos o casi identicos que difieren solo en valores de variables.

```bash
# Crear y cambiar a un workspace de desarrollo
terraform workspace new dev

# Crear uno de produccion
terraform workspace new prod

# Listar todos los workspaces
terraform workspace list

# Seleccionar uno
terraform workspace select dev
```

La limitacion principal radica en que workspaces no impone ningun tipo de restriccion sobre quien puede seleccionar哪一个 workspace esta activo. No hay control de acceso a nivel de workspace y un error humano puede provocar que se apliquen cambios sobre el entorno equivocado. Ademas, la gestion de workspaces se complica cuando los entornos requieren configuraciones significativamente distintas, ya que no hay forma de variar archivos de modulo o provedores entre workspaces sin recurrir a condicionales complejos.

> Los workspaces son adecuados para prototipos rapidos o cuando los entornos son practicamente identicos. En entornos de produccion con requisitos de seguridad estrictos, se recomienda combinarlos con otras estrategias de aislamiento.

## Directorios: carpetas separadas por entorno

La estrategia mas comunmente adoptada y recomendada consiste en organizar el codigo en una estructura de directorios donde cada entorno tiene su propia carpeta con sus archivos `.tf`, sus variables y su estado independiente. Esta separacion fisica del codigo ofrece claridad sobre que pertenece a que entorno y permite que cada carpeta mantenga su propio backend con su propio estado.

```
infraestructura/
├── dev/
│   ├── main.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   └── backend.tf
├── staging/
│   ├── main.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   └── backend.tf
├── prod/
│   ├── main.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   └── backend.tf
└── modules/
    ├── networking/
    ├── compute/
    └── database/
```

Cada carpeta de entorno referencia los mismos modulos compartidos desde `modules/`, pero configura sus propios valores de variables y backend. Esto significa que `dev/main.tf` y `prod/main.tf` pueden ser practicamente identicos, pero apuntar a diferentes buckets S3 o tablas DynamoDB para el bloqueo de estado. La ventaja de esta aproximacion es que un ingeniero puede observar la estructura del proyecto y entender inmediatamente como se organiza la infraestructura, ademas de que los controles de acceso pueden implementarse a nivel de carpeta en el repositorio.

```hcl
# dev/backend.tf
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state-dev"
    key            = "infraestructura/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-lock-dev"
  }
}

# prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state-prod"
    key            = "infraestructura/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-lock-prod"
  }
}
```

> Esta estrategia se considera la mejor practica para la mayoria de organizaciones. Permite un balance entre claridad, control de acceso granular y facilidad de automatizacion a traves de pipelines CI/CD.

## Repositorios separados: aislamiento maximo

Cuando los requisitos de seguridad o de gobernanza son estrictos, la mejor opcion es separar el codigo de cada entorno en repositorios distintos. Esta aproximacion maximiza el aislamiento porque cada repositorio puede tener sus propios permisos, sus propios revisiones de codigo y su propia politica de despliegue. Un desarrollador puede tener acceso de escritura al repositorio de dev pero solo de lectura al de produccion.

```
git@github.com:mi-empresa/terraform-dev.git
git@github.com:mi-empresa/terraform-staging.git
git@github.com:mi-empresa/terraform-prod.git
```

El inconveniente es la duplicacion de codigo y la dificultad para mantener modulos sincronizados entre repositorios. Para mitigar esto se recurre a modulos publicados como paquetes versionados o a un repositorio dedicado de modulos compartidos que cada entorno consume como dependencia. La complejidad operativa aumenta significativamente, por lo que esta estrategia se reserva para organizaciones con multiples equipos o con requisitos regulatorios que exigen segregacion estricta de responsabilidades.

> Antes de adoptar repositorios separados, evaluar si los mismos objetivos se alcanzan con directorios separados combinados con politicas de control de acceso en el repositorio monolitico y permisos de IAM restrictivos en el backend de estado.

