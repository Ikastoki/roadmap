# Uso de Modulos

Un modulo no aporta valor hasta que se instancia. Terraform ofrece multiples mecanismos para referenciar e incluir modulos en una configuracion, cada uno disenado para un origen distinto: el sistema de archivos local, el Registry publico, repositorios Git o almacenamiento empaquetado. La eleccion del origen determina como se distribuye el modulo, como se versiona y como se integra en el flujo de trabajo del equipo.

## Fuente local

El metodo mas directo consiste en referenciar un modulo mediante una ruta del sistema de archivos. La clave `source` acepta rutas relativas o absolutas que apuntan al directorio del modulo. Es el mecanismo ideal durante el desarrollo inicial, cuando el modulo aun no esta maduro para publicarse o cuando se quiere mantener el codigo dentro del mismo repositorio.

```hcl
module "vpc" {
  source = "./modules/vpc"

  nombre      = "desarrollo"
  cidr_bloque = "10.0.0.0/16"
}

module "base_datos" {
  source = "../shared/modulos/rds"

  nombre = "inventario"
  tamano = 20
}
```

La ruta se interpreta siempre desde el directorio del modulo que contiene el bloque `module`. No importa desde donde se ejecute terraform; la referencia es relativa al archivo `.tf` en el que aparece. Terraform resuelve la ruta en el momento de `terraform init` y copia el contenido del directorio fuente al cache local (`.terraform/modules/`).

El uso de rutas locales no incorpora versionado intrinseco. El modulo que se utiliza es siempre el que esta en el sistema de archivos en ese momento, lo que obliga a gestionar las versiones mediante el control de versiones del repositorio principal. Si el modulo cambia, quien lo usa obtiene automaticamente la nueva version al actualizar su rama.

## Terraform Registry

El Registry (`registry.terraform.io`) es el repositorio oficial de modulos publicos. Alberga tanto modulos mantenidos por HashiCorp como por la comunidad y proveedores cloud. La URL de un modulo en el Registry sigue la convencion de tres partes: `<namespace>/<nombre>/<proveedor>`.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "mi-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
}
```

Los modulos del Registry son el equivalente a las librerias en un gestor de paquetes como npm o PyPI. Alguien ha resuelto ya un problema comun --desplegar un VPC, configurar un cluster EKS, montar un ALB-- y lo ha empaquetado como un modulo publico. Usarlo evita reinventar la rueda y, al estar mantenido por multiples contribuyentes, suele incorporar casos borde y buenas practicas que un equipo tardaria en descubrir por si mismo.

El argumento `version` es obligatorio en el Registry (aunque no lo exige la sintaxis, es una mala practica omitirlo). Sin una restriccion de version, `terraform init` descargaria siempre la ultima disponible, lo que puede introducir cambios inesperados en la infraestructura existente. El versionado sigue el esquema semantico: `MAJOR.MINOR.PATCH`.

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"  # cualquier 19.x.x pero no 20.x.x

  cluster_name = "produccion"
  subnet_ids   = module.vpc.subredes_ids
}
```

El operador `~>` es especialmente util porque fija la version mayor permitiendo actualizaciones de menor y parche, que son retrocompatibles.

## Git y otros origenes

Cuando un modulo no esta publicado en el Registry o se quiere controlar exactamente que commit se utiliza, Terraform permite referenciar directamente un repositorio Git. Esto es comun en equipos que mantienen sus propios modulos internos en repositorios privados.

```hcl
module "autoscaling" {
  source = "git::https://github.com/mi-organizacion/terraform-modulo-autoscaling.git"

  max_tamano = 5
  min_tamano = 2
}
```

La sintaxis general sigue el formato `git::<url>`. Terraform clona el repositorio durante `terraform init` y utiliza el contenido del directorio raiz (o la ruta especificada con `//`). Se pueden refinar las referencias con argumentos adicionales:

```hcl
# Rama especifica
source = "git::https://github.com/org/modulo.git?ref=feature/nueva-red"

# Tag concreto
source = "git::https://github.com/org/modulo.git?ref=v1.2.0"

# Commit SHA fijo
source = "git::https://github.com/org/modulo.git?ref=abc123def456"

# Subdirectorio dentro del repositorio
source = "git::https://github.com/org/modulo.git//ruta/al/modulo"
```

El parametro `ref` acepta cualquier referencia que Git entienda: ramas, tags o commits. En produccion es recomendable usar tags o commits en lugar de ramas, porque una rama es un objetivo movil que cambiaria el modulo en cada `terraform init` sin que el equipo lo revise explicitamente.

Terraform tambien soporta otros origenes como URLs HTTP que apuntan a archivos comprimidos:

```hcl
module "vpc" {
  source = "https://ejemplo.com/modulos/vpc.zip"
}
```

> Las URLs HTTP son utiles para distribucion simple, pero carecen de las garantias de integridad y versionado que ofrecen Git y el Registry. Verificar el checksum del archivo descargado es responsabilidad del usuario.

## Versionado de modulos

El versionado no es un atributo intrinseco del modulo, sino de la forma en que se referencia. Cada origen impone su propio mecanismo:

- **Local**: la version es la del repositorio principal. No hay restriccion independiente.
- **Registry**: el argumento `version` con sintaxis semantica y operadores de rango (`>=`, `~>`, `<=`).
- **Git**: el parametro `ref` con tags, commits o ramas.

Independientemente del origen, Terraform almacena una copia local de cada modulo en el directorio `.terraform/modules/`. Cuando se ejecuta `terraform init` y el modulo ya existe en cache, Terraform solo lo descarga de nuevo si la referencia ha cambiado. Para forzar la recarga de todos los modulos se usa `terraform init -upgrade`:

```bash
terraform init -upgrade
```

> `terraform init -upgrade` actualiza las fuentes de todos los modulos, ignorando la cache local. Es especialmente util despues de cambiar la referencia de una rama de Git o cuando se sospecha que el Registry puede tener una version mas reciente del modulo compatible con el rango especificado.

La eleccion del mecanismo de versionado debe alinearse con el flujo de trabajo del equipo. Si el modulo se desarrolla internamente, Git con tags semanticos ofrece el mayor control. Si se consume un modulo de la comunidad, el Registry con restriccion de version mayor proporciona un equilibrio entre estabilidad y actualizaciones.

