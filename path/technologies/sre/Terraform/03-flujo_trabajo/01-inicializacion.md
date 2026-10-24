# Inicializacion del proyecto

`terraform init` es el primer comando que se ejecuta en cualquier proyecto de Terraform y, aunque su ejecucion es trivial, el trabajo que realiza internamente es fundamental para el resto del ciclo de vida. Este comando prepara el entorno de trabajo descargando los plugins necesarios, inicializando el backend donde se almacenara el estado y configurando la estructura interna que Terraform utilizara para operar. Sin una inicializacion correcta, ningun otro comando funcionara, lo que convierte a este paso en la puerta de entrada obligada a cualquier flujo de trabajo con Terraform.

## Estructura del proyecto

Un proyecto basico de Terraform comienza con un directorio de trabajo vacio donde se colocaran los archivos de configuracion. Aunque Terraform no impone una estructura rigida, la convencion mas extendida consiste en un archivo principal llamado `main.tf` que contiene la configuracion central. A medida que el proyecto crece, es habitual dividir la configuracion en multiples archivos (`variables.tf`, `outputs.tf`, `providers.tf`), ya que Terraform carga automaticamente todos los archivos con extension `.tf` del directorio actual.

```bash
mkdir proyecto-terraform
cd proyecto-terraform
touch main.tf
```

No es necesario crear archivos de configuracion adicionales para ejecutar `terraform init`, pero si es imprescindible que al menos exista un archivo `.tf` con un bloque `terraform` o `provider` para que el comando sepa que descargar.

## Ejecucion de terraform init

Una vez creado el archivo `main.tf` con el contenido minimo, se ejecuta `terraform init` desde el directorio del proyecto. Este comando es idempotente: puede ejecutarse multiples veces sin efectos secundarios, y Terraform se encarga de detectar que ya esta inicializado y omitir los pasos redundantes.

```bash
terraform init
```

La salida del comando muestra las distintas fases de la inicializacion:

```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.0.0...
- Installed hashicorp/aws v5.0.0 (signed by HashiCorp)
Terraform has been successfully initialized!
```

## Descarga de providers

Uno de los trabajos principales de `terraform init` es la descarga de los providers declarados en la configuracion. Los providers son plugins binarios que Terraform utiliza para comunicarse con las APIs de los servicios en la nube. Cada provider se descarga desde el Terraform Registry y se almacena en un directorio oculto llamado `.terraform/` dentro del proyecto.

```hcl
# Contenido minimo de main.tf para que init descargue el provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

Los providers se almacenan con una estructura de directorios que incluye la plataforma y la arquitectura del sistema, lo que permite que el mismo proyecto funcione en diferentes sistemas operativos sin conflictos.

> Terraform calcula un hash de los providers descargados y lo almacena en el archivo `.terraform.lock.hcl`. Este archivo debe versionarse en el repositorio para garantizar que todos los miembros del equipo utilicen exactamente las mismas versiones de los providers.

## Inicializacion del backend

El backend es el mecanismo que Terraform utiliza para almacenar el archivo de estado (`terraform.tfstate`). Por defecto, Terraform configura un backend local que guarda el estado en el propio directorio de trabajo. Sin embargo, en entornos colaborativos se recomienda utilizar un backend remoto como S3, Azure Storage o Terraform Cloud.

```hcl
# Configuracion de backend remoto en S3
terraform {
  backend "s3" {
    bucket = "mi-estado-terraform"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Cuando se configura un backend remoto, `terraform init` pregunta si se desea migrar el estado existente desde el backend local al remoto. Esta migracion es automatica y transparente, pero requiere confirmacion del usuario.

## Directorio .terraform

Despues de ejecutar `terraform init`, aparece el directorio `.terraform/` con la siguiente estructura:

```
.terraform/
├── providers/
│   └── registry.terraform.io/
│       └── hashicorp/
│           └── aws/
│               └── 5.0.0/
│                   └── linux_amd64/
│                       └── terraform-provider-aws
├── modules/
└── terraform.tfstate
```

Este directorio no debe versionarse (esta en `.gitignore`) porque contiene binarios especificos de la plataforma. Cada miembro del equipo debe ejecutar `terraform init` en su maquina para obtener los providers adecuados para su sistema operativo.

> Si se modifican las versiones de los providers o la configuracion del backend, es necesario ejecutar `terraform init -upgrade` para forzar la actualizacion de los providers descargados. Sin la opcion `-upgrade`, Terraform solo actualiza los providers que faltan, no los ya descargados.

