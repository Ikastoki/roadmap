# Destruccion y limpieza

La destruccion controlada de infraestructura es una de las capacidades mas valiosas de la IaC y una de las mas subestimadas. Mientras que la creacion de recursos es el objetivo visible, la capacidad de eliminarlos de forma completa, ordenada y predecible es lo que permite la verdadera agilidad operativa. Entornos efimeros de pruebas, laboratorios de validacion, despliegues de staging y escenarios de disaster recovery dependen todos de un mecanismo de destruccion confiable que deje la cuenta del proveedor libre de recursos y costos.

## Comando de destruccion

El motor recorre el estado de la infraestructura gestionada y genera un plan inverso: cada recurso marcado para eliminacion en el orden opuesto al de creacion, respetando las dependencias para evitar errores de referencias cruzadas. Primero se eliminan los recursos dependientes (instancias, bases de datos) y al final los recursos base (VPCs, grupos de seguridad, politicas IAM).

```bash
# Terraform
terraform destroy

# OpenTofu
tofu destroy

# Destruccion con confirmacion automatica (CI/CD)
terraform destroy -auto-approve
```

La salida sigue la misma estructura que `apply` pero con la operacion `-` (destroy) en todos los recursos:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_instance.web will be destroyed
  - resource "aws_instance" "web" {
      - ami           = "ami-0c55b159cbfafe1f0" -> null
      - instance_type = "t3.small" -> null
      - id            = "i-0123456789abcdef" -> null
      ...
    }

  # aws_subnet.publica will be destroyed
  - resource "aws_subnet" "publica" {
      - cidr_block              = "10.0.1.0/24" -> null
      - id                      = "subnet-067890abcdef" -> null
      ...
    }

  # aws_vpc.principal will be destroyed
  - resource "aws_vpc" "principal" {
      - cidr_block           = "10.0.0.0/16" -> null
      - id                   = "vpc-0a1b2c3d4e5f" -> null
      ...
    }

Plan: 0 to add, 0 to change, 3 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Except in very simple cases, it is recommended
  to use a targeted destroy instead.

  Enter a value: yes

aws_instance.web: Destroying... [id=i-0123456789abcdef]
aws_instance.web: Destruction complete after 40s
aws_subnet.publica: Destroying... [id=subnet-067890abcdef]
aws_subnet.publica: Destruction complete after 2s
aws_vpc.principal: Destroying... [id=vpc-0a1b2c3d4e5f]
aws_vpc.principal: Destruction complete after 3s

Destroy complete! Resources: 3 destroyed.
```

> La destruccion no es reversible. A diferencia de `apply`, donde un error puede dejar recursos parcialmente creados, `destroy` opera con la presuncion de que el usuario ha evaluado las consecuencias. En produccion, esta operacion debe requerir aprobacion explicita y no debe ejecutarse nunca sin un plan previo.

## Destruccion selectiva

En lugar de eliminar toda la infraestructura, es posible apuntar a recursos especificos para destruccion. Esto es util cuando se gestiona un recurso particular sin afectar al resto del entorno:

```bash
# Destruir solo una instancia concreta
terraform destroy -target=aws_instance.web

# Destruir un modulo completo
terraform destroy -target=module.monitoring

# En Pulumi
pulumi destroy --target urn:pulumi:dev::proyecto::aws:ec2:instance::web
```

La destruccion selectiva es una herramienta poderosa pero debe usarse con precaucion. El resultado puede dejar recursos en el estado que referencian recursos ya eliminados, generando errores en ejecuciones futuras. Despues de una destruccion selectiva, conviene ejecutar `plan` para verificar que el estado es consistente.

## Entornos efimeros y gestion de costos

Uno de los casos de uso mas significativos de la destruccion controlada es la gestion de costos en entornos de prueba y desarrollo. Las organizaciones crean periodicamente entornos completos para validaciones, demos, pruebas de carga o reproduccion de bugs, y los eliminan inmediatamente despues de su uso. Este patron, conocido como *ephemeral environments*, elimina el problema de los recursos olvidados que generan facturas inesperadas.

El ciclo tipico de un entorno efimero es:

```bash
# Crear entorno de pruebas
terraform init
terraform apply -var="environment=ephemeral-pr-42" -auto-approve

# ... ejecutar pruebas, demos, validaciones ...

# Destruir entorno completo
terraform destroy -var="environment=ephemeral-pr-42" -auto-approve
```

En entornos con costos por segundo o por minuto (functions serverless, contenedores ephemeral, clusters k8s), la destruccion inmediata despues de su uso puede reducir significativamente la factura mensual. Algunas organizaciones implementan *auto-shutdown policies* que destruyen automaticamente entornos despues de un tiempo de inactividad:

```yaml
# Ejemplo de politica de auto-destruccion con cron
# (ejecutado via pipeline programada)
schedule:
  - cron: "0 22 * * *"   # Todos los dias a las 22:00
    action: destroy
    filter: "environment LIKE 'ephemeral-%'"
```

> En equipos grandes, es comun que existan entornos efimeros generados por pull requests que nadie reclama. Establecer un TTL (time-to-live) automatico para estos entornes es una practica esencial de higiene de costos. Herramientas como Spacelift o env0 gestionan esta funcionalidad nativamente.

## Limpieza del estado

Despues de ejecutar `destroy`, el archivo de estado queda vacio de recursos pero persiste como archivo. En entornos con backend local, el archivo `terraform.tfstate` puede eliminarse manualmente. En backends remotos, el estado se mantiene pero sin recursos gestionados, lo cual es funcionalmente equivalente a un proyecto nuevo.

```bash
# Verificar que no quedan recursos en el estado
terraform state list
# Salida vacia = no hay recursos

# Eliminar el archivo de estado local (solo si se usa backend local)
rm terraform.tfstate terraform.tfstate.backup
```

Si se utiliza un modulo compartido o un estado que otros proyectos referencian, la eliminacion total del estado puede causar errores en esas dependencias. En estos casos, es mejor mantener el estado con su estructura vacia y documentar que el proyecto ha sido desactivado.

