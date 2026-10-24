# Integracion con CI/CD

El valor real de Terraform se multiplica cuando su ejecucion se integra en un pipeline de integracion continua y despliegue continuo (CI/CD). En lugar de ejecutar comandos manualmente desde una terminal, cada cambio en el repositorio dispara automaticamente un plan, se somete a revision y, si se aprueba, se aplica. Este flujo elimina el factor humano como fuente de error, garantiza que toda modificacion pasa por los mismos controles y proporciona un registro de auditoria completo de quien cambio que y cuando.

## GitHub Actions

GitHub Actions ofrece una integracion directa con Terraform mediante la accion oficial `hashicorp/setup-terraform`, que instala la version deseada del binario en el runner. El flujo tipico consta de varios trabajos (jobs) que ejecutan `terraform fmt`, `terraform validate`, `terraform plan` y, finalmente, `terraform apply` con una aprobacion manual opcional.

```yaml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

env:
  TF_VERSION: "1.6.0"

jobs:
  format-and-validate:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./infra
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      - run: terraform fmt -check
      - run: terraform init
      - run: terraform validate

  plan:
    needs: format-and-validate
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./infra
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      - run: terraform init
      - run: terraform plan -out=plan.tfplan
      - uses: actions/upload-artifact@v4
        with:
          name: terraform-plan
          path: ./infra/plan.tfplan

  apply:
    if: github.ref == 'refs/heads/main'
    needs: plan
    runs-on: ubuntu-latest
    environment: production
    defaults:
      run:
        working-directory: ./infra
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      - uses: actions/download-artifact@v4
        with:
          name: terraform-plan
          path: ./infra
      - run: terraform init
      - run: terraform apply plan.tfplan
```

El uso de `environment: production` en el trabajo de apply permite activar la proteccion de entorno de GitHub: el trabajo queda pausado hasta que un responsable autoriza la ejecucion, proporcionando el punto de aprobacion manual necesario antes de desplegar en produccion.

Las credenciales del proveedor cloud se gestionan mediante OpenID Connect (OIDC) o secrets de GitHub. OIDC es la opcion recomendada: elimina la necesidad de almacenar claves de acceso de larga duracion y permite roles IAM federados con GitHub Actions.

## GitLab CI

GitLab CI incluye plantillas oficiales para Terraform en el registro de GitLab, accesibles mediante la palabra clave `include`. La integracion aprovecha el modelo de entornos de GitLab (environments) y los recursos protegidos (protected environments) para controlar el despliegue en produccion.

```yaml
# .gitlab-ci.yml
image: hashicorp/terraform:1.6

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/infra
  TF_PLAN_NAME: plan.tfplan

cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - ${TF_ROOT}/.terraform

before_script:
  - cd ${TF_ROOT}
  - terraform init

stages:
  - validate
  - plan
  - apply

validate:
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -out=${TF_PLAN_NAME}
  artifacts:
    paths:
      - ${TF_ROOT}/${TF_PLAN_NAME}
    expire_in: 30 days

apply:
  stage: apply
  script:
    - terraform apply ${TF_PLAN_NAME}
  only:
    - main
  environment:
    name: production
  when: manual
```

El bloque `environment: production` combinado con `when: manual` crea un trabajo que solo se ejecuta en la rama principal, requiere activacion manual y, si se configura como entorno protegido, restringe quien puede desplegar. GitLab muestra ademas el estado del entorno y el historial de despliegues en la interfaz web.

Los recursos de GitLab CI como variables de grupo o variables de proyecto almacenan las credenciales cloud. Cada variable puede marcarse como protegida (solo disponible en ramas protegidas) y enmascarada en los logs.

## Jenkins

Jenkins requiere una configuracion mas explicita que las plataformas SaaS, pero ofrece un control total sobre cada aspecto del pipeline. El enfoque habitual es un Jenkinsfile declarativo que define stages paralelos para `validate` y `plan`, y un stage secuencial para `apply` con aprobacion manual.

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        TF_DIR = "${WORKSPACE}/infra"
    }

    stages {
        stage('Validate') {
            steps {
                dir(env.TF_DIR) {
                    sh 'terraform init'
                    sh 'terraform fmt -check'
                    sh 'terraform validate'
                }
            }
        }
        stage('Plan') {
            steps {
                dir(env.TF_DIR) {
                    sh "terraform plan -out=plan.tfplan"
                }
            }
            post {
                success {
                    stash name: 'plan', includes: "**/plan.tfplan"
                }
            }
        }
        stage('Apply') {
            when {
                branch 'main'
                expression { currentBuild.result == null }
            }
            input {
                message "Aprobar apply en produccion?"
                ok "Aplicar"
            }
            steps {
                dir(env.TF_DIR) {
                    unstash 'plan'
                    sh 'terraform apply plan.tfplan'
                }
            }
        }
    }
}
```

El bloque `input` en el stage de Apply introduce una pausa en el pipeline que muestra un dialogo de aprobacion en la interfaz de Jenkins. Solo cuando un usuario con permisos suficientes hace clic en "Aprobar", el pipeline continua con el apply. Jenkins permite ademas roles y permisos granulares mediante el plugin Role-Based Strategy, restringiendo quien puede aprobar despliegues en produccion frente a entornos de desarrollo.

Los plugins recomendados para Terraform en Jenkins incluyen el plugin oficial de Terraform (que gestiona la instalacion del binario por version) y Credentials Binding para inyectar credenciales cloud de forma segura en las variables de entorno del build.

> Tanto Jenkins como GitHub Actions y GitLab CI comparten el mismo patron fundamental: validar antes de planificar, planificar antes de aplicar, y requerir aprobacion humana para entornos sensibles. La diferencia radica en la sintaxis de configuracion y en el modelo de permisos de cada plataforma.

