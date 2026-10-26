# Ejecucion remota de IaC

Ejecutar comandos de Terraform o Pulumi desde la maquina de un desarrollador introduce un problema fundamental: la infraestructura depende del estado local de ese equipo, de las credenciales almacenadas en su disco y de la version exacta del CLI que tenga instalada. Esto genera incongruencias dificiles de reproducir, donde dos personas pueden obtener resultados distintos al aplicar el mismo conjunto de archivos de configuracion. La ejecucion remota resuelve este problema delegando la planificacion y aplicacion de cambios a un servidor gestionado o un servicio SaaS que mantiene un entorno controlado, un estado centralizado y un registro de auditoria completo de cada operacion.

Un servidor de ejecucion remota actua como un agente intermediario: recibe los archivos HCL o YAML del repositorio, ejecuta los comandos correspondientes en un entorno aislado con versiones predefinidas de los providers y el CLI, y almacena el state en un backend remoto. Esto elimina la necesidad de que cada desarrollador configure credenciales de cloud en su maquina local y garantiza que todas las ejecuciones se realizan con la misma version de las herramientas. La centralizacion del state, ademas, permite que multiples personas trabajen en la misma infraestructura sin bloquearse mutuamente, ya que el servicio gestiona locks automaticos durante las operaciones de escritura.

## Terraform Cloud

Terraform Cloud es el servicio gestionado de HashiCorp que implementa este modelo. Cada workspace representa un conjunto de configuraciones Terraform vinculadas a un repositorio de codigo fuente, y ejecuta automaticamente los comandos `plan` y `apply` en respuesta a cambios en el repositorio.

```hcl
# workspace.tf - Configuracion del workspace en Terraform Cloud
terraform {
  cloud {
    organization = "mi-empresa"

    workspaces {
      name = "produccion-infra"
    }
  }
}
```

La configuracion del workspace en la interfaz web o mediante la API define que ejecutables se disparan y que variables estan disponibles:

```yaml
# workspace-settings.yaml (API representation)
workspace:
  name: produccion-infra
  execution-mode: remote
  terraform-version: "1.9.0"
  working-directory: environments/prod
  vcs-repo:
    identifier: mi-empresa/infra-repo
    branch: main
    ingress-submodules: false
  variables:
    - key: AWS_REGION
      value: eu-west-1
      category: env
    - key: AWS_ACCESS_KEY_ID
      value: vault://secrets/aws/prod/access-key
      category: env
      sensitive: true
    - key: AWS_SECRET_ACCESS_KEY
      value: vault://secrets/aws/prod/secret-key
      category: env
      sensitive: true
```

Cada push al repositorio o cada comentario con `/plan` en un pull request desencadena una ejecucion remota. El resultado del plan se registra y se puede revisar desde la interfaz web, incluyendo el diff exacto de recursos que seran creados, modificados o destruidos.

> Terraform Cloud ofrece dos modos de ejecucion: `remote` (ejecucion completa en la nube de HashiCorp) y `agent` (ejecucion en un agente que tu gestionas dentro de tu propia VPC). El modo agent es necesario cuando los recursos requieren conectividad a redes privadas o cuando las politicas de seguridad prohiben que el estado salga de tu infraestructura.

## Spacelift y env0

Spacelift y env0 son alternativas que aportan funcionalidades adicionales como la gestion de dependencias entre stacks, la integracion con politicas OPA (Open Policy Agent) y soporte nativo para Pulumi y CloudFormation ademas de Terraform.

```yaml
# Spacelift stack definition
stack:
  name: produccion-infra
  repository: mi-empresa/infra-repo
  branch: main
  project_root: environments/prod
  autoredeploy: false
  triggers:
    - path: environments/prod/**
      pattern: push
  labels:
    - produccion
    - aws
  runner:
    image: spacelift/runner:1.9.0
  policy:
    attach:
      - spacelift-apply-check
      - cost-estimation-policy
```

```yaml
# env0 project configuration
project:
  name: infraestructura-core
  repository:
    url: https://github.com/mi-empresa/infra-repo
    branch: main
    path: environments/staging
  revision: HEAD
  terraform_version: "1.9.0"
  requires_approval: true
  auto_deploy_on_push: false
  environment_variables:
    - name: TF_VAR_region
      value: eu-west-1
  credentials:
    - name: aws-production
      type: cloud
      value_from:
        vault_secret: prod/aws/credentials
```

## Atlantis

Atlantis adopta un enfoque diferente: se ejecuta como un servidor self-hosted dentro de tu infraestructura y se integra con GitHub, GitLab o Bitbucket mediante webhooks. En lugar de reemplazar la ejecucion local, Atlantis intercepta los pull requests y ejecuta los comandos de Terraform dentro de comentarios del propio PR.

```yaml
# atlantis.yaml - Configuracion del repositorio Atlantis
version: 3
projects:
  - name: produccion-vpc
    dir: environments/prod/vpc
    workspace: default
    autoplan:
      enabled: true
      when_modified: ["*.tf", "*.tfvars", "modules/**/*.tf"]
      enabled: true
    apply_requirements: [approved, mergeable]
    workflow: custom

  - name: produccion-eks
    dir: environments/prod/eks
    workspace: default
    depends_on:
      - produccion-vpc
    autoplan:
      enabled: false
    apply_requirements: [approved]

workflows:
  custom:
    plan:
      steps:
        - init
        - run: terraform validate -no-color
        - plan:
            extra_args: ["-var-file=environments/prod/terraform.tfvars"]
    apply:
      steps:
        - apply
```

La configuracion del servidor Atlantis define los repositorios autorizados y las reglas de acceso:

```yaml
# server-config.yaml
repos:
  - id: /.*/
    apply_requirements: [approved, mergeable, undiscussed]
    allow_custom_workflows: true
    allowed_overrides: [apply_requirements]
    allow_private_modules: true

hosts:
  - hostname: github.com
    type: github
    org: mi-empresa
    webhook_url: https://atlantis.mi-empresa.com/events
    webhook_secret: ${ATLANTIS_WEBHOOK_SECRET}

```

> Atlantis es especialmente util en equipos que prefieren un modelo de revision donde todo el ciclo de vida de la infraestructura se documenta en el pull request: el plan aparece como comentario, las aprobaciones se gestionan con los mecanismos nativos de GitHub y la aplicacion se ejecuta mediante un comando escrito en el propio PR.

## Comparativa de enfoques

Cada modelo de ejecucion remota tiene implicaciones distintas sobre la gobernanza y la experiencia de desarrollo. Terraform Cloud y Spacelift centralizan completamente la ejecucion en sus plataformas, lo que simplifica la operacion pero genera dependencia de un proveedor externo. Atlantis mantiene el control dentro de la organizacion pero requiere infraestructura adicional para alojar el servidor. La eleccion depende del equilibrio entre la necesidad de control interno y la capacidad operativa del equipo para mantener la herramienta.

| Caracteristica | Terraform Cloud | Spacelift / env0 | Atlantis |
|---|---|---|---|
| Ejecucion | Servidor gestionado | Servidor gestionado | Self-hosted |
| Estado | Backend centralizado | Backend centralizado | Backend configurado manualmente |
| Aprobaciones | Nativo | Nativo + OPA | PR review nativo |
| Coste | SaaS por seat | SaaS por seat | Infraestructura propia |

