# Planificacion y preview

La planificacion es el mecanismo de seguridad fundamental que separa la intencion del usuario de la ejecucion real sobre la infraestructura. Antes de que cualquier llamada a una API de cloud provider se realize, el motor de IaC construye un grafo completo de los cambios necesarios, clasificandolos por tipo y evaluando sus implicaciones. Este proceso de dry-run permite al humano revisar, validar y aprobar cada operacion antes de que tenga efectos reales, eliminando la posibilidad de cambios inesperados o destructivos.

## Comparacion de estados

El motor de planificacion realiza una comparacion triangular entre tres representaciones de la infraestructura: el **estado conocido** (que es lo que el motor cree que existe, almacenado en el archivo de estado), la **configuracion deseada** (que es lo que el usuario ha escrito en los archivos `.tf` o equivalentes) y la **realidad real** (que es lo que efectivamente existe en el provider). La capacidad de detectar desviaciones entre el estado conocido y la realidad real se denomina *drift detection* y es una de las ventajas criticas de la IaC sobre la gestion manual.

```
                    ┌─────────────────┐
                    │  Configuracion  │
                    │    Deseada      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Planificador  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼──────┐ ┌────▼─────┐ ┌──────▼───────┐
     │ Estado Conocido│ │  Reality │ │  Drift       │
     │  (tfstate)     │ │  Check   │ │  Detection   │
     └───────────────┘ └──────────┘ └──────────────┘
```

Cuando se ejecuta el comando de planificacion, el motor:

1. Lee el estado actual (local o remoto).
2. Hace *refresh* del estado contra el provider real para detectar cambios manuales o de otros procesos.
3. Compara el estado resultante con la configuracion deseada.
4. Genera un plan de ejecucion con las operaciones necesarias.

```bash
# Terraform
terraform plan -out=plan.bin

# OpenTofu
tofu plan -out=plan.bin
```

> Siempre usar el flag `-out=plan.bin` (o cualquier nombre de archivo) para guardar el plan en un archivo binario. Este archivo garantiza que lo revisado por el humano es exactamente lo que se ejecutara en `apply`, evitando que cambios en la configuracion entre `plan` y `apply` alteren el resultado.

## Generacion del plan de ejecucion

El plan de ejecucion clasifica cada recurso en tres categorias que se representan con prefijos visuales en la salida del comando:

- **`+` (create):** El recurso no existe en el estado ni en el provider; sera creado.
- **`~` (update):** El recurso existe pero una o mas propiedades cambiaron; sera modificado in-place.
- **`-` (destroy):** El recurso existe en el estado pero no en la configuracion; sera eliminado.
- **`-+` (replace):** El recurso tiene cambios que obligan a destruirlo y recrearlo (por ejemplo, cambio de immutable attribute).
- **`<=` (read):** Una fuente de datos sera consultada sin crear ni modificar nada.

Ejemplo de salida de planificacion:

```
Terraform will perform the following actions:

  # aws_vpc.principal will be created
  + resource "aws_vpc" "principal" {
      + cidr_block           = "10.0.0.0/16"
      + enable_dns_hostnames = true
      + enable_dns_support   = true
      + id                   = (known after apply)
      + tags                 = {
          + "Environment" = "production"
          + "ManagedBy"   = "terraform"
        }
    }

  # aws_subnet.publica will be created
  + resource "aws_subnet" "publica" {
      + availability_zone       = "eu-west-1a"
      + cidr_block              = "10.0.1.0/24"
      + id                      = (known after apply)
      + map_public_ip_on_launch = true
      + vpc_id                  = (known after apply)
    }

  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
      ~ instance_type = "t3.micro" -> "t3.small"
        id            = "i-0123456789abcdef"
      + tags          = {
          + "Name" = "web-production"
        }
    }

  # aws_olvidado.legacy will be destroyed
  - resource "aws_olvidado" "legacy" {
      - id   = "i-abcdef0123456789" -> null
      - name = "servidor-antiguo" -> null
    }

Plan: 2 to add, 1 to change, 1 to destroy.
```

La linea final del resumen (`Plan: 2 to add, 1 to change, 1 to destroy`) proporciona una vista inmediata del alcance del cambio. En un entorno de produccion, cualquier cambio clasificado como destroy o replace debe examinarse con especial atencion.

## Planificacion en Pulumi

Pulumi ejecuta una planificacion equivalente internamente, aunque la interfaz de usuario difiere. El comando `preview` genera un dry-run que muestra los cambios proyectados:

```bash
pulumi preview --diff
```

La salida de Pulumi es similar en estructura, mostrando cada recurso con su operacion proyectada. La diferencia principal es que Pulumi resuelve las dependencias de forma imperativa (el lenguaje de programacion ejecuta secuencialmente) mientras que Terraform y OpenTofu las resuelven de forma declarativa a traves del grafo de dependencias.

> En Pulumi, el equivalente a `-out` es la captura del plan con `--diff` y revision manual antes de `pulumi up`. No existe un archivo binario de plan, pero el motor garantiza que lo mostrado en `preview` es exactamente lo que ejecutara si el usuario confirma.

## Revision como paso obligatorio

La planificacion no es simplemente un paso tecnico; es el momento en que el ingeniero ejerce juicio profesional. Antes de aprobar un plan, es necesario verificar:

- **Que no hay cambios inesperados.** Si el plan muestra mas operaciones de las esperadas, puede haber dependencias implicitas o archivos de estado desactualizados.
- **Que las sustituciones son inevitables.** Un `-+` (replace) en un recurso con datos persistentes (una base de datos, por ejemplo) puede causar perdida de datos si no se gestiona correctamente.
- **Que los valores sensibles aparecen marcados.** Si un output o atributo sensible se muestra en texto plano, hay que revisar que las protecciones de `sensitive` estan configuradas.
- **Que el orden de operaciones es correcto.** El motor determina el orden automaticamente, pero en escenarios complejos (actualizaciones blue-green, migraciones de esquema) puede ser necesario intervenir con `depends_on` o `lifecycle` blocks.

```
Plan: 2 to add, 1 to change, 1 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: 
```

> En automatizaciones CI/CD, el plan se ejecuta como parte del pipeline y su salida se registra como artefacto. La aprobacion puede ser automatica (si el plan solo contiene creaciones) o manual (si contiene modificaciones o destrucciones en entornos productivos). Herramientas como Atlantis o Spacelift gestionan este flujo con revision basada en pull requests.

