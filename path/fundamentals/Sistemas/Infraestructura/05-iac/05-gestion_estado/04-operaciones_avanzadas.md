# Operaciones Avanzadas con el Estado

La gestion del estado mas alla de las operaciones basicas de plan y apply permite resolver situaciones complejas que surgen en ciclos de vida reales de infraestructura: recursos creados manualmente que deben ser incorporados al codigo, recursos que necesitan ser removidos del estado sin eliminarlos fisicamente, y revisiones detalladas de la composicion actual del estado para depuracion o auditoria. Estas operaciones son criticas para mantener la consistencia entre el codigo declarativo y la realidad operativa del entorno.

El comando `terraform state list` proporciona una vision completa de todos los recursos que Terraform esta gestionando actualmente. Esta informacion es fundamental para identificar recursos especificos antes de ejecutar operaciones de importacion o eliminacion del estado, y para verificar que no existen recursos duplicados o huérfanos que puedan causar conflictos en futuras ejecuciones.

La importacion de recursos es el proceso de tomar un recurso que fue creado manualmente o por otra herramienta y registrarla en el estado de Terraform sin modificar el recurso real. Esto permite que Terraform asuma el control de la gestion futura del recurso, incluyendo actualizaciones de configuracion, monitoreo de drift y eliminacion controlada.

## Inspeccion del estado

El listado de recursos en el estado permite identificar rapidamente que esta siendo gestionado por Terraform y cuales son los identificadores internos que la herramienta utiliza para cada recurso.

```bash
# Listar todos los recursos en el estado
terraform state list

# Listar recursos de un tipo especifico
terraform state list aws_instance.*

# Listar recursos con un patron de nombre
terraform state list aws_iam_role.*

# Mostrar detalles de un recurso especifico
terraform state show aws_instance.web_server
```

```json
# Salida formateada de terraform show (JSON)
{
  "mode": "managed",
  "type": "aws_instance",
  "name": "web_server",
  "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
  "instances": [
    {
      "schema_version": 1,
      "attributes": {
        "id": "i-0abc123def456789",
        "ami": "ami-0c55b159cbfafe1f0",
        "instance_type": "t3.micro",
        "private_ip": "10.0.1.15",
        "public_ip": "54.210.123.45"
      }
    }
  ]
}
```

> El comando `terraform state pull` descarga el estado completo como JSON, util para auditorias o para procesarlo con herramientas externas como `jq` para generar reportes personalizados.

## Eliminacion de recursos del estado

El comando `terraform state rm` elimina un recurso del estado sin destruirlo en el proveedor cloud. Esto es util cuando un recurso debe ser gestionado por otro equipo, otra herramienta de IaC, o cuando se necesita excluir temporalmente un recurso de la gestion de Terraform sin afectar su disponibilidad.

```bash
# Eliminar un recurso especifico del estado
terraform state rm aws_instance.web_server

# Eliminar un modulo completo del estado
terraform state rm module.vpc

# Eliminar un recurso con count o for_each
terraform state rm 'aws_instance.web_server[0]'
terraform state rm 'aws_instance.web_server["produccion"]'
```

```hcl
# Despues de eliminar del estado, se recomienda agregar una exclusion en el codigo
# para evitar que Terraform intente recrear el recurso en el proximo plan

# Opcion 1: Comentar el bloque de recurso
# resource "aws_instance" "web_server" {
#   ami           = "ami-0c55b159cbfafe1f0"
#   instance_type = "t3.micro"
# }

# Opcion 2: Usar lifecycle ignore_changes
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  lifecycle {
    ignore_changes = [tags]
  }
}
```

> La eliminacion de recursos del estado debe ejecutarse con cuidado. Si el recurso sigue definido en el codigo fuente, el proximo `terraform plan` mostrara que el recurso necesita ser creado, lo cual podria causar un conflicto si el recurso original sigue activo con el mismo nombre.

## Importacion de recursos existentes

La importacion permite incorporar recursos que ya existen en el proveedor cloud bajo la gestion de Terraform. Este proceso requiere definir el bloque de recurso en el codigo fuente con la configuracion que refleje el estado actual del recurso, y luego ejecutar el comando de importacion para registrar la relacion en el estado.

```bash
# Importar un recurso existente
terraform import aws_instance.web_server i-0abc123def456789

# Importar un recurso con count
terraform import 'aws_instance.web_server[0]' i-0abc123def456789

# Importar un recurso con for_each
terraform import 'aws_instance.web_server["produccion"]' i-0abc123def456789

# Importar un modulo completo (Terraform >= 1.5)
terraform import -var-file=production.tfvars module.vpc
```

```hcl
# Bloque de recurso necesario para la importacion
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"  # Debe coincidir con el AMI real
  instance_type = "t3.micro"                # Debe coincidir con el tipo real
  
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}

# Despues de importar, ejecutar terraform plan para verificar consistencia
# Si hay diferencias, ajustar el bloque de recurso para que coincida exactamente
```

```bash
# Flujo completo de importacion
# 1. Definir el bloque de recurso en el codigo
# 2. Ejecutar la importacion
terraform import aws_instance.web_server i-0abc123def456789

# 3. Verificar que no hay diferencias no deseadas
terraform plan

# Si el plan muestra cambios, ajustar el bloque de recurso
# hasta que el plan muestre "No changes"
terraform plan
```

> La importacion no modifica el recurso existente en el proveedor. Terraform simplemente registra en el estado que el recurso con el ID especificado corresponde al bloque de recurso definido en el codigo. Cualquier diferencia entre el bloque de recurso y el estado real del proveedor se detectara en el siguiente plan.

## Renombrar recursos en el estado

El comando `terraform state mv` permite cambiar el identificador de un recurso en el estado sin modificar el recurso real en el proveedor. Esto es util para refactorizar el codigo, mover recursos entre modulos o corregir errores de denominacion.

```bash
# Renombrar un recurso
terraform state mv aws_instance.web_server aws_instance.application_server

# Mover un recurso a un modulo
terraform state mv aws_instance.web_server module.web.aws_instance.server

# Mover un recurso con count
terraform state mv 'aws_instance.web_server[0]' 'aws_instance.application_server[0]'
```

> Al renombrar recursos, es imperativo actualizar simultaneamente el bloque de recurso en el codigo fuente. Si el estado referencia un nombre que no existe en el codigo, Terraform eliminara el recurso en el proximo plan.

