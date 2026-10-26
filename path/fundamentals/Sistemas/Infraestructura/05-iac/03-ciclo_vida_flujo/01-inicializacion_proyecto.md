# Inicializacion de proyecto

Todo proyecto de infraestructura como codigo comienza con una fase de inicializacion que establece el entorno de trabajo, resuelve dependencias y configura el mecanismo de almacenamiento del estado. Este proceso no solo descarga proveedores y modulos necesarios, sino que tambien crea la estructura interna que permetira al motor de IaC comparar el estado actual con el deseado en cada ejecucion posterior. Sin una inicializacion correcta, ninguno de los comandos subsiguientes (`plan`, `apply`, `destroy`) podra ejecutarse, ya que el tooling carecera de los componentes binarios y la configuracion de backend requeridos.

## Directorio de trabajo

La convencion universal consiste en dedicar un repositorio (o un subdirectorio dentro de el) a cada conjunto de configuraciones relacionadas. El directorio de trabajo determina implicitamente el alcance del proyecto: todos los archivos `.tf`, `*.tf.json` o `Pulumi.yaml` que se encuentren en el directorio actual (y sus subdirectorios) seran procesados por el motor de IaC. Mantener un directorio de trabajo limpio, sin archivos de configuracion de otros proyectos, evita que el motor intente resolver recursos y variables que no corresponden al contexto actual.

```
mi-proyecto-iac/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── backend.tf
└── modules/
    ├── red/
    └── computo/
```

> Es recomendable inicializar el proyecto dentro de un repositorio versionado con Git. El directorio `.terraform/` o `.opentofu/` generado tras la inicializacion contiene binarios y metadatos locales; debe agregarse al `.gitignore`.

## Descarga de dependencias

Durante la inicializacion, el motor resuelve las dependencias declaradas en los archivos de configuracion. Esto incluye la descarga de proveedores (binarios que implementan la interfaz de comunicacion con APIs de cloud providers), modulos reutilizables y plugins adicionales. Los binarios se almacenan en un directorio local oculto (`.terraform/` en Terraform, `.opentofu/` en OpenTofu) organizados por plataforma y arquitectura.

En Terraform y OpenTofu, el comando de inicializacion es practicamente identico:

```bash
# Terraform
terraform init

# OpenTofu
tofu init
```

La salida del comando muestra la descarga de cada componente:

```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Finding hashicorp/tls versions matching "~> 4.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)
- Installing hashicorp/tls v4.0.5...
- Installed hashicorp/tls v4.0.5 (signed by HashiCorp)

Terraform has been successfully initialized!
```

Los proveedores se descargan desde el [Terraform Registry](https://registry.terraform.io/) por defecto, aunque se pueden configurar registries privados. Cada proveedor firma sus releases criptograficamente, y el motor verifica esta firma durante la instalacion para garantizar la integridad del binario.

## Configuracion del backend

El backend determina donde se almacena el estado de la infraestructura. Por defecto, Terraform y OpenTofu utilizan un archivo local (`terraform.tfstate`), pero en entornos colaborativos es imprescindible configurar un backend remoto que permita el bloqueo de estado y el trabajo concurrente. La configuracion del backend se define en un bloque dedicado y se activa durante la inicializacion.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mi-estado-terraform"
    key            = "produccion/infraestructura.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

> Cuando se modifica la configuracion del backend, ejecutar `terraform init -migrate-state` para migrar el estado existente al nuevo backend sin perder informacion.

Otros backends soportados incluyen Azure Blob Storage, Google Cloud Storage, HashiCorp Consul y PostgreSQL. La caracteristica comun a todos ellos es el soporte para **bloqueo de estado**: cuando un usuario ejecuta `plan` o `apply`, el backend adquiere un lock que impide que otro usuario modifique el estado simultaneamente, evitando condiciones de carrera.

## Inicializacion en Pulumi

Pulumi adopta un modelo diferente. En lugar de un directorio oculto local, el estado se almacena en un servicio gestionado por defecto (Pulumi Cloud) o en un backend de almacenamiento configurado. La inicializacion en Pulumi se realiza mediante el comando `login`, que establece la conexion con el backend de estado, seguido de la instalacion de dependencias del proyecto.

```bash
# Login al backend de estado de Pulumi
pulumi login s3://mi-bucket-pulumi-estado

# Instalacion de dependencias (para proyectos con provider nativo)
pulumi install
```

Para proyectos que utilizan proveedores SDK (como `@pulumi/aws`), las dependencias se gestiona igual que en cualquier proyecto del ecosistema correspondiente:

```bash
# Node.js
npm install @pulumi/aws @pulumi/pulumi

# Python
pip install pulumi pulumi-aws

# Go (modulos)
go mod tidy
```

## Comandos complementarios

Existen variantes y opciones del comando de inicializacion que resultan utiles en escenarios especificos:

```bash
# Reconectar al backend sin reinstalar proveedores
terraform init -backend=true -upgrade

# Actualizar proveedores a sus ultimas versiones compatibles
terraform init -upgrade

# Inicializacion sin configurar backend (util para plan local rapido)
terraform init -backend=false
```

El flag `-upgrade` fuerza la busqueda de versiones mas recientes de los proveedores dentro de las restricciones de version definidas. Esto es particularmente util cuando se mantiene un proyecto a largo plazo y se desea incorporar correcciones de seguridad y mejoras de rendimiento de los proveedores.

