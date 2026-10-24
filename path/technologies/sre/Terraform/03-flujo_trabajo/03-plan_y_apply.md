# Planificacion y aplicacion

Los comandos `terraform plan` y `terraform apply` constituyen el ciclo central del flujo de trabajo de Terraform. El primero permite inspeccionar los cambios que se realizarian sin aplicarlos, actuando como un mecanismo de validacion y revision. El segundo ejecuta esos cambios y materializa la infraestructura declarada en el codigo. Separar estas dos fases es una de las decisiones de diseno mas acertadas de Terraform, ya que introduce un punto de control humano entre la intencion y la ejecucion.

## terraform plan

`terraform plan` compara la configuracion actual con el estado almacenado y genera una descripcion detallada de las operaciones necesarias para alcanzar el estado deseado. Este comando no modifica ningun recurso real; es una operacion de solo lectura que consulta el estado y muestra las diferencias.

```bash
terraform plan
```

La salida muestra tres categorias de cambios:

- **Crear**: recursos que existen en la configuracion pero no en el estado. Se muestran con el prefijo `+`.
- **Modificar**: recursos que existen tanto en la configuracion como en el estado pero cuyos atributos difieren. Se muestran con el prefijo `-/+` (destruir y recrear) o `~` (actualizar in-place).
- **Destruir**: recursos que existen en el estado pero no en la configuracion. Se muestran con el prefijo `-`.

```hcl
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                          = "ami-0c55b159cbfafe1f0"
      + instance_type                = "t2.micro"
      + id                           = (known after apply)
      + subnet_id                    = (known after apply)
    }

  # aws_security_group.web will be created
  + resource "aws_security_group" "web" {
      + id                           = (known after apply)
      + name                         = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

Los valores marcados como `(known after apply)` son atributos que Terraform no puede conocer hasta que el recurso se haya creado, como el ID asignado por el proveedor cloud. Terraform gestiona estas incognitas durante el plan y las resuelve durante el apply.

## Dry-run y revision

El plan actua como un dry-run que permite al equipo revisar que cambios se van a realizar antes de ejecutarlos. Esta separacion es especialmente valiosa en entornos de produccion, donde un cambio incorrecto podria provocar una interrupcion del servicio. En un flujo de CI/CD tipico, `terraform plan` se ejecuta en un pull request y sus resultados se revisan antes de aprobar el merge.

```bash
# Guardar el plan en un archivo binario para aplicarlo despues
terraform plan -out=plan.tfplan

# Ver un plan guardado sin ejecutarlo
terraform show plan.tfplan
```

El archivo de plan generado con `-out` contiene toda la informacion necesaria para aplicar los cambios, incluyendo las incognitas resueltas. Esto permite separar la generacion del plan (en un entorno de CI) de su ejecucion (en un entorno con privilegios), garantizando que se aplica exactamente lo que se reviso.

> La opcion `-out` es especialmente util en pipelines de CI/CD. El plan se genera durante la validacion del codigo y se almacena como artefacto. El `apply` posterior utiliza ese mismo archivo, asegurando que no se han introducido cambios entre la revision y la ejecucion.

## terraform apply

`terraform apply` ejecuta las operaciones descritas en el plan para alcanzar el estado deseado. Si se ejecuta sin un archivo de plan, Terraform vuelve a calcular el plan y solicita confirmacion interactiva al usuario. Si se proporciona un archivo de plan, la ejecucion es automatica y no requiere confirmacion.

```bash
# Con confirmacion interactiva
terraform apply

# Sin confirmacion (automatizado)
terraform apply -auto-approve

# Usando un archivo de plan previamente generado
terraform apply plan.tfplan
```

Durante la ejecucion, Terraform muestra el progreso de cada recurso:

```
aws_security_group.web: Creating...
aws_security_group.web: Creation complete after 2s [id=sg-12345678]
aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]
aws_instance.web: Creation complete after 30s [id=i-0a1b2c3d4e5f67890]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

Terraform maneja automaticamente las dependencias entre recursos. Si un recurso depende de otro (por ejemplo, una instancia EC2 depende de un grupo de seguridad), Terraform espera a que el recurso dependiente se cree completamente antes de crear el que lo referencia. Esta espera es posible gracias al grafo de dependencias que Terraform construye durante el analisis de la configuracion.

## Generacion y actualizacion del estado

Al finalizar `terraform apply`, Terraform actualiza el archivo de estado (`terraform.tfstate`) con los datos reales de los recursos creados. Este archivo contiene un mapeo completo entre los recursos definidos en la configuracion y los objetos reales en el proveedor cloud, incluyendo atributos como IDs, ARNs, IPs y cualquier otro dato devuelto por la API.

```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "i-0a1b2c3d4e5f67890",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t2.micro"
          }
        }
      ]
    }
  ]
}
```

El estado es el unico punto de verdad que Terraform utiliza para saber que recursos existen. Si se pierde el estado y no existe una copia de seguridad, Terraform perdera la nocion de los recursos que gestiona, aunque estos sigan existiendo en el proveedor cloud. Por eso es fundamental almacenar el estado en un backend remoto con respaldos automaticos.

> No se debe editar el archivo de estado manualmente. Cualquier modificacion directa puede provocar inconsistencias entre el estado y los recursos reales. Para operaciones avanzadas sobre el estado existen comandos especificos como `terraform state rm`, `terraform state mv` o `terraform import`.

