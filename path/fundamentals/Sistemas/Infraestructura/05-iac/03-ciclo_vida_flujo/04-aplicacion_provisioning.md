# Aplicacion y provisioning

La fase de aplicacion es donde la configuracion declarativa se traduce en llamadas reales a las APIs de los providers de infraestructura. El motor toma el plan de ejecucion aprobado (o genera uno nuevo si no se proporciono un archivo de plan guardado) y lo ejecuta secuencialmente, respetando el orden de dependencias calculado durante la planificacion. Cada operacion implica una serie de peticiones HTTP a los servicios cloud: crear un recurso, esperar su disponibilidad, actualizar sus propiedades, eliminarlo. Tras cada operacion exitosa, el archivo de estado se actualiza para reflejar la nueva realidad de la infraestructura.

## Ejecucion del plan aprobado

El comando basico de aplicacion acepta un archivo de plan binario, lo que garantiza que se ejecuta exactamente lo que se reviso durante la planificacion:

```bash
# Terraform: aplicar plan pre-guardado
terraform apply plan.bin

# Terraform: generar y aplicar en un solo paso (sin revision previa)
terraform apply

# OpenTofu
tofu apply plan.bin
tofu apply
```

Cuando se ejecuta `apply` sin argumentos, el motor genera un nuevo plan en el momento y solicita confirmacion interactiva antes de proceder. Esta practica es aceptable en entornos de desarrollo pero desaconsejada en produccion, donde el flujo correcto implica planificar con `-out`, revisar el plan guardado y luego aplicarlo con el archivo binario.

Salida tipica de un `terraform apply`:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
  - destroy

Terraform will perform the following actions:

  # aws_vpc.principal will be created
  + resource "aws_vpc" "principal" {
      + cidr_block           = "10.0.0.0/16"
      + enable_dns_hostnames = true
      + enable_dns_support   = true
      + id                   = (known after apply)
      ...
    }

Plan: 2 to add, 1 to change, 1 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_vpc.principal: Creating...
aws_vpc.principal: Creation complete after 3s [id=vpc-0a1b2c3d4e5f]
aws_subnet.publica: Creating...
aws_subnet.publica: Creation complete after 1s [id=subnet-067890abcdef]
aws_instance.web: Modifying... [id=i-0123456789abcdef]
aws_instance.web: Modifications complete after 45s [id=i-0123456789abcdef]
aws_olvidado.legacy: Destroying... [id=i-abcdef0123456789]
aws_olvidado.legacy: Destruction complete after 30s

Apply complete! Resources: 2 added, 1 changed, 1 destroyed.

Outputs:

vpc_id           = "vpc-0a1b2c3d4e5f"
subnet_publica_id = "subnet-067890abcdef"
instance_public_ip = "52.16.34.56"
```

## Que ocurre internamente durante la aplicacion

El motor de IaC no ejecuta las operaciones en paralelo de forma arbitraria; sigue un grafo de dependencias topologicamente ordenado. Los recursos sin dependencias entre si se crean simultaneamente, maximizando la velocidad de despliegue. Los recursos que dependen de otros esperan a que estos se creen completamente antes de iniciar su propia creacion.

El ciclo de vida de cada operacion sobre un recurso sigue este patron:

1. **Invocacion del provider:** El motor envia una peticion a la API del provider (por ejemplo, `CreateVPC`, `RunInstances`) con los parametros definidos en la configuracion.
2. **Espera de estado:** La mayoria de los recursos cloud son asincronos. La API retorna inmediatamente un identificador, y el motor realiza *polling* periodicamente para verificar cuando el recurso alcanza el estado deseado (available, running, active).
3. **Escritura de estado:** Una vez confirmado que el recurso existe y esta funcional, el motor actualiza el archivo de estado con los metadatos del recurso (ID, ARN, atributos calculados por el provider).
4. **Continuacion:** El motor procede al siguiente recurso en el grafo de dependencias.

> El timeout por defecto para la creacion de recursos es tipicamente de 10 minutos. Si un recurso tarda mas (una base de datos RDS con snapshot, por ejemplo), es necesario configurar explicitamente el `timeouts` block. Si el timeout se alcanza, el motor cancela la operacion y marca el recurso como en estado de error.

## Actualizacion del estado

El archivo de estado es el registro maestro de la infraestructura gestionada. Cada operacion exitosa lo actualiza con la informacion mas reciente. El estado contiene:

- **Identificadores unicos** de cada recurso en el provider (ARNs, IDs, URIs).
- **Atributos computados** que el provider retorna y que no estan en la configuracion (IPs, puertos, fechas de creacion).
- **Dependencias implicitas** detectadas automaticamente por el motor.
- **Metadatos** de la propia ejecucion (timestamp, version del provider).

En colaboraciones con backend remoto, el estado se bloquea durante `apply` para evitar que otro usuario ejecute cambios simultaneos que pudieran generar conflictos. El lock se libera automaticamente al completar la operacion o al producirse un error, aunque en casos excepcionales puede ser necesario liberarlo manualmente:

```bash
# Liberar un lock stale (usar con precaucion)
terraform force-unlock LOCK_ID
```

## Aplicacion en Pulumi

Pulumi ejecuta el equivalente funcional de `apply` mediante `up`, que combina planificacion y ejecucion en un unico comando:

```bash
pulumi up
```

El motor de Pulumi genera el plan, lo muestra al usuario y ejecuta si se confirma. La diferencia conceptual es que Pulumi mantiene un checkpoint interno en el backend de estado despues de cada operacion, permitiendo un reinicio parcial si la ejecucion se interrumpe a mitad.

```bash
# Pulumi con auto-approve (CI/CD, sin interaccion)
pulumi up --yes --non-interactive
```

> En flujos de CI/CD, tanto `terraform apply -auto-approve plan.bin` como `pulumi up --yes` eliminan la interaccion humana. La aprobacion debe gestionarse en la capa de orquestacion (pull request approvals, pipelines con gates), no en la herramienta de IaC.

## Errores y manejo de fallos

Cuando una operacion falla (timeout, error de permisos, cota excedida), el motor detiene la ejecucion y reporta el error. Los recursos creados exitosamente antes del fallo **no se deshacen automaticamente**. Esto es intencional: revertir parcialmente podria dejar la infraestructura en un estado inconsistente. El ingeniero debe decidir si:

- Corregir la configuracion y volver a ejecutar `apply` (el motor reintentara solo la operacion fallida).
- Ejecutar `terraform destroy -target=RECURSO_ESPECIFICO` para eliminar recursos parciales no deseados.
- En casos criticos, revertir manualmente los cambios desde la consola del provider.

