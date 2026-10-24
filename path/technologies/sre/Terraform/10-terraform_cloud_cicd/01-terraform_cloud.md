# Terraform Cloud

Terraform Cloud es la plataforma SaaS de HashiCorp que profesionaliza el uso de Terraform en equipos y organizaciones. No se trata de una herramienta distinta, sino de un entorno gestionado que resuelve los problemas que surgen al escalar Terraform mas alla de una unica maquina local: comparticion de estado, ejecucion centralizada, control de acceso granular y gobernanza sobre las configuraciones. Actua como un backend remoto que no solo almacena el estado, sino que tambien ejecuta los planes y applies, manteniendo un registro historico completo de cada operacion.

## Remote State y Remote Execution

El problema mas inmediato que resuelve Terraform Cloud es la gestion del estado en equipos. Con un backend local o incluso con S3, cada miembro del equipo debe configurar manualmente el acceso al estado y coordinarse para evitar operaciones simultaneas. Terraform Cloud centraliza el estado en su infraestructura, proporcionando versionado automatico, bloqueo nativo y un historial de cambios consultable desde la interfaz web.

La ejecucion remota (remote execution) va un paso mas alla: en lugar de ejecutar `terraform plan` y `terraform apply` en la maquina local, Terraform Cloud las ejecuta en sus propios runners gestionados. Esto elimina la dependencia del entorno local del desarrollador: las credenciales, versiones de Terraform y variables de entorno se configuran una unica vez en la plataforma y se reutilizan en todas las ejecuciones. El flujo de trabajo tipico consiste en ejecutar `terraform plan` desde el CLI local, que envia la configuracion a Terraform Cloud, este calcula el plan en sus servidores y devuelve el resultado. Si se aprueba, `terraform apply` se ejecuta tambien remotamente.

```bash
# Configurar el backend en main.tf
terraform {
  cloud {
    organization = "mi-organizacion"
    workspaces {
      name = "infraestructura-prod"
    }
  }
}

# Inicializar y autenticar
terraform init
terraform login
terraform plan   # Se ejecuta en remoto
```

> La ejecucion remota implica que el CLI de Terraform actua como un cliente ligero. La maquina local solo necesita el binario de Terraform y un token de API; las credenciales de cloud (AWS, Azure, GCP) se almacenan como variables en Terraform Cloud.

## Workspaces en la Nube

Terraform Cloud hereda el concepto de workspaces de Terraform open source y lo extiende con capacidades organizativas. Un workspace en Terraform Cloud agrupa una configuracion de Terraform, su estado asociado, las variables de entorno y las variables de Terraform. Cada workspace puede apuntar a un repositorio Git diferente o a una rama especifica, permitiendo asociar entornos completos (desarrollo, staging, produccion) a ramas de codigo.

Cada workspace expone una interfaz web con informacion detallada: el ultimo estado conocido, el historial de runs, los outputs actuales y los recursos gestionados. Las variables se dividen en dos categorias: variables de Terraform (las definidas con `variable` en el codigo) y variables de entorno (que se inyectan como variables de sistema, utiles para credenciales de provider). Ambas pueden marcarse como sensibles, lo que oculta su valor en la interfaz y en los logs.

```hcl
# Las variables se definen en el codigo de forma habitual
variable "region" {
  description = "Region de AWS"
  type        = string
}

variable "instance_type" {
  description = "Tipo de instancia"
  type        = string
  default     = "t3.micro"
}
```

En Terraform Cloud, los valores se asignan desde la interfaz web o la API, y las variables marcadas como sensibles (como `AWS_SECRET_ACCESS_KEY`) se encriptan en reposo y nunca se muestran en texto plano.

## Sentinel y Políticas como Código

Sentinel es el framework de politicas como codigo de HashiCorp, integrado nativamente en Terraform Cloud. Permite definir restricciones programaticas sobre los planes de Terraform antes de que se apliquen, actuando como una capa de guarda entre el plan y la ejecucion. Esto es especialmente valioso en organizaciones donde multiples equipos despliegan infraestructura y se necesita garantizar el cumplimiento de normas internas sin depender de la buena voluntad de cada desarrollador.

Una politica Sentinel se escribe en un lenguaje especifico (Sentinel Language) y se evalua contra el plan de Terraform. La politica puede pasar (se permite la ejecucion), fallar (se bloquea) o adherirse (se permite pero se registra una advertencia). Los policiesets agrupan politicas relacionadas y se asocian a workspaces concretos o a toda una organizacion.

```sentinel
# Ejemplo: politica Sentinel que exige etiquetas obligatorias
import "tfplan/v2" as tfplan

required_tags = ["environment", "owner", "cost-center"]

all_resources = filter tfplan.resource_changes as _, rc {
    rc.mode == "managed" and rc.type != "terraform_data"
}

violations = filter all_resources as _, resource {
    tags = resource.requires["tags"]
    not all required_tags as tag {
        tag in tags
    }
}

main = rule {
    length(violations) == 0
}
```

Las politicas se evaluan en tres fases del ciclo de vida: antes del plan (para validar configuracion), tras el plan (para validar cambios previstos) y antes del apply (para validar estado final). Este modelo permite, por ejemplo, bloquear recursos de tipos no autorizados, exigir etiquetas de cumplimiento, limitar regiones permitidas o asegurar que ciertos recursos tengan encriptacion habilitada.

## Planes y Costos

Terraform Cloud ofrece distintos planes que determinan las funcionalidades disponibles. El plan gratuito incluye hasta cinco usuarios, un unico agente de ejecucion concurrente y funcionalidades basicas como remote state y remote execution. Los planes de pago (Team, Governance, Enterprise) anaden control de acceso basado en roles (RBAC), politicas Sentinel, agentes auto-gestionados (para entornos sin acceso directo a internet), auditoria avanzada y soporte SLA.

La eleccion del plan depende del tamano del equipo y de los requisitos de gobernanza. Para un equipo pequeno que busca centralizar el estado y la ejecucion, el plan gratuito es suficiente. Para organizaciones que necesitan politicas de cumplimiento, multiples equipos con separacion de responsabilidades y auditoria detallada, los planes Team o Governance se vuelven necesarios.

> Una alternativa auto-gestionada es Terraform Enterprise, que despliega la funcionalidad completa de Terraform Cloud en infraestructura propia, util en sectores regulados donde los datos no pueden salir del centro de datos de la organizacion.

