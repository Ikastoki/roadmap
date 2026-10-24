# Estrategias Multi-Entorno

La necesidad de mantener entornos aislados (desarrollo, pruebas, produccion) es un requisito basico en cualquier organizacion que despliegue software. En Terraform existen tres estrategias fundamentales para abordar la gestion multi-entorno, cada una con diferentes grados de aislamiento, complejidad operativa y coste de mantenimiento. La eleccion depende del tamano del equipo, la criticidad de los entornos y la tolerancia al riesgo de la organizacion.

## Workspaces: simplicidad con un unico backend

La estrategia de workspaces, descrita en el capitulo anterior, consiste en utilizar el mismo codigo y el mismo directorio de trabajo, cambiando unicamente el workspace activo para desplegar en distintos entornos. Es el enfoque mas sencillo de implementar porque no requiere reestructurar el repositorio ni duplicar configuraciones.

```
proyecto/
  main.tf
  variables.tf
  outputs.tf
  terraform.tfvars
```

Un unico comando `terraform workspace select dev && terraform apply` despliega en desarrollo; el mismo comando con `prod` despliega en produccion. Todo el equipo comparte el mismo backend remoto y los estados se diferencian por el nombre del workspace.

**Ventajas:**
- Minimalismo estructural: un unico conjunto de archivos, un unico `terraform init`.
- Facil incorporacion: cualquier miembro del equipo entiende el flujo en minutos.
- Bajo coste de infraestructura de estado: un unico bucket o unica organizacion de Terraform Cloud.

**Inconvenientes:**
- Aislamiento debil: todos los entornos comparten backend, proveedor y credenciales. Un error al seleccionar workspace puede destruir produccion.
- Dificultad para tener configuraciones divergentes: si los entornos requieren diferencias sustanciales (distinta region, distinto proveedor, distinta version de modulo), el codigo se llena de condicionales.
- Arranque en frio: cambiar de workspace implica reconstruir el grafo de recursos desde cero en el nuevo workspace, lo que en proyectos grandes puede ser lento.

> Los workspaces son adecuados para equipos pequenos, proyectos monoliticos o entornos no criticos donde la velocidad de desarrollo prima sobre el aislamiento. HashiCorp recomienda workspaces para diferenciar *entornos* dentro del mismo *proyecto*, no para gestionar proyectos completamente distintos.

## Directorios separados: aislamiento estructural

La segunda estrategia organiza el repositorio con directorios independientes para cada entorno, cada uno con su propio conjunto de archivos de configuracion y su propio estado. Es el enfoque mas utilizado en la industria por su equilibrio entre aislamiento y mantenibilidad.

```
proyecto/
  modules/
    vpc/
    compute/
    database/
  environments/
    dev/
      main.tf
      variables.tf
      backend.tf
      dev.tfvars
    staging/
      main.tf
      variables.tf
      backend.tf
      staging.tfvars
    prod/
      main.tf
      variables.tf
      backend.tf
      prod.tfvars
```

Cada directorio de entorno es un proyecto Terraform independiente. Se inicializa por separado (`terraform init` en cada uno), tiene su propio archivo de estado y puede usar backends distintos si es necesario. La reutilizacion se consigue mediante modulos compartidos ubicados en `modules/`.

```hcl
# environments/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"
  cidr   = "10.0.0.0/16"
  env    = "prod"
}

module "compute" {
  source      = "../../modules/compute"
  instance_type = "t3.large"
  env         = "prod"
}
```

**Ventajas:**
- Aislamiento completo: cada entorno tiene su propio backend, sus propias credenciales y su propio ciclo de vida.
- Configuraciones divergentes sin condicionales: los entornos pueden usar diferentes modulos, diferentes versiones o diferentes proveedores sin contaminar el codigo de los demas.
- Seguridad granular: se pueden asignar permisos IAM distintos a cada directorio, restringiendo el acceso a produccion.

**Inconvenientes:**
- Duplicacion de configuracion: aunque los modulos se compartan, cada entorno necesita su propio `main.tf`, `backend.tf` y `providers.tf`.
- Mayor numero de inicializaciones: cada directorio requiere su propio `terraform init` y ocupa su propia cache de proveedores.
- Posible deriva de configuracion: si no se revisan con disciplina, los entornos pueden acabar con configuraciones ligeramente distintas.

## Repositorios separados: maximo aislamiento

La estrategia mas radical consiste en alojar cada entorno en un repositorio Git independiente. Cada repositorio contiene todo lo necesario para desplegar ese entorno: modulos, configuraciones, pipelines y secretos.

```
github.com/org/
  terraform-modules/          # Repositorio compartido de modulos
  infrastructure-dev/         # Repositorio para dev
  infrastructure-staging/     # Repositorio para staging
  infrastructure-prod/        # Repositorio para produccion
```

Este enfoque se utiliza tipicamente en organizaciones grandes donde los equipos de plataforma son distintos de los equipos de aplicacion, o donde los requisitos de cumplimiento normativo (PCI-DSS, SOC 2, HIPAA) exigen una separacion estricta entre entornos.

**Ventajas:**
- Aislamiento total: no existe posibilidad de error humano que afecte a produccion desde un entorno de desarrollo.
- Permisos independientes: cada repositorio puede tener sus propias reglas de branch protection, sus propios secretos de CI/CD y sus propios revisores.
- Ciclos de vida desacoplados: se puede actualizar la configuracion de desarrollo sin pasar por la burocracia de produccion, y viceversa.

**Inconvenientes:**
- Fragmentacion: la configuracion se dispersa en multiples repositorios, lo que dificulta tener una vision global de la infraestructura.
- Coste de mantenimiento elevado: cambios que afectan a todos los entornos (ej. actualizar una version de provider) deben replicarse en cada repositorio.
- Mayor complejidad en CI/CD: los pipelines deben coordinarse entre repositorios, y la promocion de cambios de un entorno a otro requiere integraciones adicionales.

## Criterios de seleccion

No existe una estrategia universalmente superior. La decision se alinea con la estructura del equipo y la criticidad del sistema:

| Criterio | Workspaces | Directorios separados | Repositorios separados |
|---|---|---|---|
| Tamano del equipo | 1-3 personas | 3-15 personas | 15+ personas |
| Riesgo de error humano | Alto | Medio | Bajo |
| Coste operativo | Bajo | Medio | Alto |
| Velocidad de cambio | Alta | Media | Baja |
| Cumplimiento normativo | Bajo | Medio | Alto |
| Complejidad del proyecto | Baja | Media | Alta |

La mayoria de organizaciones encuentran en la estrategia de directorios separados el punto optimo entre aislamiento y simplicidad. Los workspaces quedan relegados a prototipos, laboratorios personales o equipos muy reducidos. Los repositorios separados se justifican unicamente cuando el riesgo o la regulacion lo exigen.

