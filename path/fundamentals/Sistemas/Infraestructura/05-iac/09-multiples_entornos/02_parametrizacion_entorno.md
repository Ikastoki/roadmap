# Parametrizacion por entorno

Una vez establecida la estrategia de aislamiento, el siguiente desafio tecnico consiste en que cada entorno pueda configurarse de forma independiente sin duplicar el codigo fuente. Terraform resuelve este problema mediante un sistema de variables y archivos de valores por defecto que permite inyectar configuraciones especificas en cada ejecucion. La parametrizacion efectiva no solo determina que valores toman las variables, sino tambien como se seleccionan automaticamente durante el ciclo de vida del despliegue, especialmente en pipelines de integracion y despliegue continuo donde la intervencion manual debe ser minima.

## Archivos tfvars por entorno

Terraform carga automaticamente el archivo `terraform.tfvars` si existe en el directorio de ejecucion, pero tambien permite especificar archivos adicionales con el flag `-var-file`. Este mecanismo es ideal para mantener un archivo de valores por entorno que se pasa explicitamente en el comando de plan o apply.

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Nombre del entorno (dev, staging, prod)"
}

variable "instance_type" {
  type        = string
  description = "Tipo de instancia EC2"
}

variable "instance_count" {
  type        = number
  description = "Numero de instancias a desplegar"
  default     = 1
}

variable "enable_monitoring" {
  type        = bool
  description = "Habilitar monitoreo avanzado"
  default     = false
}
```

```hcl
# dev.tfvars
environment     = "dev"
instance_type   = "t3.micro"
instance_count  = 2
enable_monitoring = false
```

```hcl
# prod.tfvars
environment     = "prod"
instance_type   = "m5.large"
instance_count  = 5
enable_monitoring = true
```

```bash
# Ejecutar plan para desarrollo
terraform plan -var-file="dev.tfvars"

# Ejecutar apply para produccion
terraform apply -var-file="prod.tfvars"
```

> Los archivos tfvars no deben contener valores sensibles como contraseñas o tokens de acceso. Para esos casos se recurre a variables de entorno o a gestores de secretos externos como Vault o AWS Secrets Manager.

## Inyeccion de variables desde CI/CD

En un pipeline automatizado, la seleccion del archivo tfvars correcto se determina automaticamente en funcion de la rama del repositorio o del contexto de ejecucion. El pipeline evalua en que rama se esta ejecutando (o que trigger activo la ejecucion) y selecciona el archivo de configuracion correspondiente sin intervencion manual.

```yaml
# Ejemplo de pipeline GitLab CI
stages:
  - plan
  - apply

variables:
  TF_ROOT: infraestructura/

.plan_template: &plan_template
  stage: plan
  script:
    - cd $TF_ROOT
    - terraform init -backend-config="backend-${ENVIRONMENT}.hcl"
    - terraform plan -var-file="${ENVIRONMENT}.tfvars" -out=tfplan
  artifacts:
    paths:
      - $TF_ROOT/tfplan

plan:dev:
  <<: *plan_template
  variables:
    ENVIRONMENT: dev
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

plan:staging:
  <<: *plan_template
  variables:
    ENVIRONMENT: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "release/*"

plan:prod:
  <<: *plan_template
  variables:
    ENVIRONMENT: prod
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual
```

```yaml
# Ejemplo de pipeline GitHub Actions
name: Terraform Deploy

on:
  push:
    branches: [develop, release/*, main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment:
          - dev
          - staging
          - prod
    steps:
      - uses: actions/checkout@v4
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      - name: Terraform Init
        run: terraform init -backend-config="backend-${{ matrix.environment }}.hcl"
        working-directory: infraestructura/
      - name: Terraform Plan
        run: terraform plan -var-file="${{ matrix.environment }}.tfvars" -out=tfplan
        working-directory: infraestructura/
```

El patron comun consiste en que cada entorno define tanto su archivo tfvars como su configuracion de backend. El backend tambien puede parametrizarse con archivos `.hcl` separados, de modo que cada entorno apunte a su propio bucket de estado y tabla de bloqueo sin necesidad de modificar archivos de codigo fuente.

```hcl
# backend-dev.hcl
bucket         = "mi-empresa-terraform-state-dev"
key            = "infraestructura/terraform.tfstate"
region         = "eu-west-1"
dynamodb_table = "terraform-lock-dev"
encrypt        = true

# backend-prod.hcl
bucket         = "mi-empresa-terraform-state-prod"
key            = "infraestructura/terraform.tfstate"
region         = "eu-west-1"
dynamodb_table = "terraform-lock-prod"
encrypt        = true
```

> La separacion entre archivos tfvars y archivos de backend permite que el codigo de Terraform sea completamente agnostico al entorno. Los unicos archivos que varian son los de configuracion especificos de cada entorno, lo que reduce significativamente la superficie de error y facilita la auditoria de cambios.

