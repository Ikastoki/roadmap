# Fuentes de Modulos

Terraform resuelve modulos desde multiples origenes, cada uno con implicaciones distintas en cuanto a gobernanza, acceso, versionado y velocidad de despliegue. La eleccion de la fuente determina quien puede consumir el modulo, como se actualiza, y que garantias ofrece en terminos de calidad y mantenimiento. Comprender estas diferencias es esencial para disenar una estrategia de modularizacion que escale con la organizacion.

La directriz `source` en un bloque `module` es el mecanismo unico que Terraform utiliza para localizar y descargar codigo de modulos. Segun el formato del valor —una ruta local, una URL de Git, o un identificador de registro— Terraform activa una estrategia de resolucion diferente. Esta uniformidad en la interfaz permite cambiar la fuente de un modulo sin alterar la configuracion que lo consume, siempre que la interfaz del modulo (variables y outputs) se mantenga estable.

## Fuentes Locales

Las fuentes locales referencian modulos que residen en el mismo repositorio o en rutas accesibles desde la maquina donde se ejecuta Terraform. Es la forma mas simple y comun durante el desarrollo inicial o en organizaciones pequenas:

```hcl
module "vpc" {
  source = "../../modules/vpc"

  cidr_block = "10.0.0.0/16"
  azs        = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}
```

Las rutas relativas (`../../modules/`) resuelven contra el directorio de trabajo actual de Terraform. Las rutas absolutas tambien son validas pero reducen la portabilidad del proyecto. Una alternativa es utilizar variables para la ruta base:

```hcl
variable "modules_path" {
  type    = string
  default = "./modules"
}

module "vpc" {
  source = "${var.modules_path}/vpc"

  cidr_block = "10.0.0.0/16"
}
```

> Las fuentes locales no soportan versionado nativo. Para aplicar control de versiones, se debe recurrir a ramas de Git, tags, o un registro externo que actue como intermediario.

## Terraform Registry

El [Terraform Registry](https://registry.terraform.io) es el registro publico principal para modulos de Terraform. Los modulos publicados aqui siguen convenciones estrictas de estructura, documentacion y versionado. La sintaxis de referencia sigue el patron `<PROVEEDOR>/<NOMBRE>/<MODULO>`:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "mi-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

El registro resuelve automaticamente la ubicacion del repositorio Git del modulo, descarga la version solicitada, y la almacena en cache local. El versionado sigue semantico: el bloque `version` acepta rangos como `~> 5.0` (parches y menores), `>= 5.0, < 6.0` (rango explicito), o `~> 5.1.0` (solo parches a partir de 5.1.0).

### Registros Privados

Las organizaciones con requisitos de gobernanza utilizan registros privados que albergan modulos internos. Terraform Cloud/Enterprise ofrece esta funcionalidad nativamente, y herramientas como Artifactory, Nexus o GitLab Package Registry pueden servir como alternativas:

```hcl
module "auth_service" {
  source  = "app.terraform.io/mi-organizacion/auth-service/aws"
  version = "2.3.1"

  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  database_url    = module.database.connection_string
}
```

Los registros privados permiten aplicar politicas de aprobacion, escaneo de seguridad y obligatoriedad de versiones antes de que un modulo este disponible para consumo.

## Repositorios Git

Los modulos pueden residir en repositorios Git accesibles —GitHub, GitLab, Bitbucket, o repositorios internos— y referenciarse directamente mediante URLs o sintaxis abreviada. Terraform clona el repositorio y utiliza la referencia solicitada (rama, tag o commit) para resolver el contenido:

```hcl
module "custom_monitoring" {
  source = "git::https://github.com/mi-organizacion/terraform-modules.git//monitoring?ref=v1.4.0"

  cluster_name = module.eks.cluster_name
  alert_email  = "ops@ejemplo.com"
}
```

La sintaxis descompone en: protocolo (`git::`), URL del repositorio, ruta relativa dentro del repositorio (`//monitoring`), y referencia de version (`?ref=v1.4.0`). Sin la referencia `ref`, Terraform utiliza la rama por defecto (tipicamente `main`), lo cual es inapropiado para produccion ya que el contenido puede cambiar en cualquier momento.

Las referencias aceptan ramas, tags y commits:

```hcl
# Por tag (recomendado para produccion)
source = "git::https://github.com/org/modules.git//vpc?ref=v2.1.0"

# Por rama (solo desarrollo)
source = "git::https://github.com/org/modules.git//vpc?ref=feature/nueva-vpc"

# Por commit (auditoria, reproduccion exacta)
source = "git::https://github.com/org/modules.git//vpc?ref=a1b2c3d"
```

> Las ramas como referencia de version son inestables por definicion: su contenido cambia con cada merge. Utilizar tags o hashes de commit es la unica forma de garantizar reproducibilidad en entornos de produccion.

### Acceso a Repositorios Privados

Los repositorios Git privados requieren autenticacion. Terraform utiliza las credenciales del sistema operativo —`git credential helper`— o puede configurarse con tokens de acceso mediante variables de entorno:

```bash
# Mediante SSH
export GIT_SSH_COMMAND="ssh -i ~/.ssh/deploy_key"

# Mediante HTTPS con token
export TF_VAR_git_token="ghp_xxxxxxxxxxxxxxxxxxxx"
```

```hcl
module "private_module" {
  source = "git::https://${var.git_token}@github.com/org/private-modules.git//database?ref=v1.0.0"
}
```

> Nunca hardcodee tokens de acceso en archivos `.tf`. Utilice variables de entorno, vault, o el sistema de variables de Terraform Cloud para inyectar credenciales de forma segura.

## S3 y Otros Backends Remotos

Terraform soporta directamente modulos alojados en S3, GCS y Azure Blob Storage. Esta opcion es util cuando la organizacion ya utiliza un bucket de almacenamiento como parte de su pipeline de CI/CD:

```hcl
module "networking" {
  source = "s3::https://mi-bucket-regional.s3.eu-west-1.amazonaws.com/modules/networking-v3.zip"

  cidr_block = "10.0.0.0/16"
}
```

El prefijo `s3::` indica a Terraform que descargue y extraiga un archivo comprimido desde la ubicacion especificada. Este enfoque requiere gestionar manualmente el versionado de los archivos ZIP en el bucket, a diferencia de los registros que manejan el versionado automaticamente.

## Estrategia de Seleccion de Fuente

La eleccion de fuente no es meramente tecnica —refleja la madurez organizacional y los requisitos de gobernanza:

| Fuentes | Uso recomendado | Versionado | Acceso |
|---------|----------------|------------|--------|
| Local | Desarrollo, prototipado | Sin control nativo | Local |
| Registry publico | Comunidad, estandares | Semantico | Publico |
| Registry privado | Organizacion | Semantico + politicas | Autenticado |
| Git | Equipos pequenos, cross-equipos | Tags/commits | SSH/HTTPS |
| S3/GCS | Integracion CI/CD existente | Manual (ZIP) | IAM |

Para organizaciones que comienzan, la recomendacion es combinar modulos locales durante el desarrollo con un registro privado para versiones estables. A medida que la base de modulos crece, la migracion a un registro con politicas de aprobacion y escaneo automatizado se vuelve indispensable.

