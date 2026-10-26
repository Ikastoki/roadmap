# Bloqueo del Estado en IaC

El bloqueo del estado es un mecanismo de exclusion mutua que impide que multiples procesos modifiquen simultaneamente el archivo de estado, protegiendo contra corrupcion de datos y aplicaciones concurrentes conflictivas. Sin este mecanismo, dos desarrolladores ejecutando `terraform apply` al mismo tiempo sobre el mismo conjunto de recursos producirian un estado inconsistente donde las actualizaciones de uno sobreescribirian las del otro, potencialmente perdiendo cambios criticos o generando recursos duplicados.

El problema fundamental es que Terraform opera bajo el supuesto de que tiene acceso exclusivo al estado durante la fase de planificacion y ejecucion. Cuando dos procesos leen el mismo estado, calculan sus planes basados en la misma imagen inicial, pero los resultados de uno no son visibles para el otro durante la ejecucion. Esto crea una carrera critica que puede resultar en recursos huérfanos, configuraciones parciales o corrupcion total del archivo de estado.

Los backends remotos implementan bloqueo utilizando primitivas de concurrencia proporcionadas por los servicios subyacentes. Amazon DynamoDB ofrece tablas con locks distribuidos, Google Cloud Storage utiliza archivos de interbloqueo, y Azure Blob Storage emplea leases bloqueados. Estos mecanismos garantizan que solo un proceso pueda obtener el lock y proceder con la modificacion del estado.

## Configuracion del lock con DynamoDB

La tabla de DynamoDB debe configurarse con una clave de particion simple que almacene el identificador del archivo de estado. Terraform utiliza esta tabla para adquirir y liberar locks de forma automatica durante cada operacion que modifique el estado.

```hcl
# Configuracion de la tabla de DynamoDB para locks
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Purpose = "Terraform state locking"
  }
}
```

```bash
# Crear la tabla de DynamoDB desde la CLI
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region eu-west-1

# Verificar que la tabla existe
aws dynamodb describe-table \
  --table-name terraform-locks \
  --region eu-west-1
```

> La tabla de DynamoDB para locking debe estar en la misma region que el bucket S3 del backend. La latencia inter-region puede causar timeouts durante operaciones de locking y desbloqueo.

## Flujo del mecanismo de bloqueo

Cuando un usuario ejecuta `terraform apply`, el motor realiza las siguientes operaciones en secuencia: primero intenta adquirir el lock en DynamoDB con un TTL especifico, luego lee el estado actual desde S3, calcula el plan de ejecucion, aplica los cambios, escribe el nuevo estado en S3, y finalmente libera el lock. Si otro proceso intenta adquirir el lock mientras el primero lo mantiene, recibe un error indicando que el estado esta bloqueado.

```
Proceso A                          Proceso B
   |                                   |
   |-- acquire lock (OK) -->           |
   |-- read state                      |
   |-- calculate plan                  |
   |                                  |-- acquire lock (BLOCKED)
   |-- apply changes                   |
   |-- write state                     |
   |-- release lock -->               |
   |                                  |-- acquire lock (OK) --> ...
```

> Si un proceso falla durante la ejecucion sin liberar el lock, el TTL del registro en DynamoDB expira automaticamente despues del periodo configurado (tipicamente 10 minutos), permitiendo que otros procesos continuen.

## Manejo de locks stuck

Cuando un proceso se interrumpe abruptamente (fallo de red, cancelacion manual, caida del servicio de CI/CD), el lock puede permanecer activo hasta que expire el TTL. Para situaciones donde es necesario liberar el lock inmediatamente, Terraform proporciona un comando de forzado.

```bash
# Ver el estado actual del lock
terraform force-unlock <LOCK_ID>

# El LOCK_ID se obtiene del mensaje de error cuando otro proceso tiene el lock
# Ejemplo de mensaje de error:
# Error: Error acquiring the state lock
# Error message: ConditionalCheckFailedException: The conditional request failed
# Lock ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Forzar el desbloqueo
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

> El comando `force-unlock` debe utilizarse con precaucion. Solo es seguro cuando se tiene certeza de que el proceso que originalmente adquirio el lock ya no esta ejecutandose. Forzar el desbloqueo durante una operacion activa puede causar corrupcion del estado.

## Configuracion de backend con locking habilitado

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state"
    key            = "produccion/infraestructura.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

```bash
# Migrar a un backend con locking habilitado
terraform init -migrate-state

# Verificar que el locking funciona
# En una terminal:
terraform plan

# En otra terminal (simultaneamente):
terraform apply
# Resultado esperado: Error acquiring the state lock
```

> En entornos de CI/CD, es recomendable configurar timeouts cortos para el lock (10-15 minutos) y monitorear las métricas de DynamoDB para detectar procesos que mantienen el lock por periodos inusuales, lo cual puede indicar un problema subyacente en el pipeline.

