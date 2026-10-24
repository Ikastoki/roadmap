# Outputs

Los outputs (salidas) son el mecanismo mediante el cual Terraform expone informacion de la infraestructura creada o gestionada. Cuando Terraform aplica una configuracion, los outputs se muestran en la salida estandar y se almacenan en el archivo de estado, lo que permite a otros sistemas, scripts o personas consumir esa informacion sin tener que inspeccionar manualmente los recursos.

Un output tipicamente expone identificadores de recursos (ARNs, IDs), direcciones IP, nombres DNS, puertos de conexion, URL de acceso o cualquier otro dato que un operador o un proceso automatizado necesite para interactuar con la infraestructura desplegada.

## Definicion de outputs

La sintaxis basica del bloque `output` es:

```hcl
output "nombre" {
  value     = <expresion>
  description = "Descripcion opcional"
  sensitive   = <true|false>
  depends_on  = [<recursos>]
}
```

El unico campo obligatorio es `value`, que contiene la expresion a exponer. El `description` es una buena practica para documentar que representa cada salida. `sensitive` controla si el valor se muestra en los logs.

```hcl
output "vpc_id" {
  description = "ID de la VPC creada"
  value       = aws_vpc.main.id
}

output "instancia_public_ip" {
  description = "IP publica de la instancia web"
  value       = aws_instance.web.public_ip
}

output "rds_endpoint" {
  description = "Endpoint de conexion a la base de datos"
  value       = aws_db_instance.main.endpoint
}
```

Los outputs se pueden definir en cualquier archivo `.tf`, pero la convencion es agruparlos en `outputs.tf` para mantener la estructura del proyecto organizada.

## Usos principales

### Informacion para el operador

El caso mas inmediato es proporcionar al usuario informacion util tras el despliegue:

```bash
$ terraform apply
...
Outputs:

vpc_id = "vpc-0a1b2c3d4e5f67890"
instancia_public_ip = "54.123.45.67"
rds_endpoint = "mydb.c9ak3n4s5k6f.us-east-1.rds.amazonaws.com:5432"
```

Sin outputs, el operador tendria que buscar estos valores manualmente en la consola AWS o mediante comandos de la CLI de AWS.

### Comunicacion entre modulos

Los outputs son el mecanismo fundamental para que los modulos hijos expongan valores a sus modulos padre. Cuando un modulo raiz utiliza un modulo hijo, puede acceder a sus outputs mediante la sintaxis `module.<nombre_modulo>.<output_name>`:

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "instancias" {
  source    = "./modules/instancias"
  vpc_id    = module.vpc.vpc_id
  subnet_ids = module.vpc.subnet_ids
}
```

### Integracion con scripts externos

Los outputs pueden ser consumidos por scripts de shell, herramientas de CI/CD o cualquier sistema que necesite informacion de la infraestructura:

```bash
# Capturar outputs en variables de shell tras el apply
VPC_ID=$(terraform output -raw vpc_id)
INSTANCE_IP=$(terraform output -raw instancia_public_ip)

# Usar la IP para ejecutar un playbook de Ansible
ansible-playbook -i "$INSTANCE_IP," deploy.yml
```

## El parametro sensitive

Cuando un output contiene informacion sensible (contrasenas, claves privadas, tokens de acceso), es necesario marcarlo con `sensitive = true` para evitar que Terraform lo muestre en la salida estandar durante `plan` y `apply`.

```hcl
output "db_password" {
  description = "Contrasena de la base de datos"
  value       = aws_db_instance.main.password
  sensitive   = true
}

output "ssh_private_key" {
  description = "Clave privada SSH para acceso a instancias"
  value       = tls_private_key.main.private_key_pem
  sensitive   = true
}
```

Con `sensitive = true`, Terraform oculta el valor en los logs:

```bash
$ terraform apply
...
Outputs:

db_password = (sensitive value)
vpc_id = "vpc-0a1b2c3d4e5f67890"
```

Para recuperar un valor sensible desde la linea de comandos, se utiliza `terraform output -raw` con el nombre del output:

```bash
terraform output -raw db_password
```

> El parametro `sensitive` no debe considerarse un mecanismo de seguridad. El valor sigue almacenandose en texto plano en el archivo de estado (`terraform.tfstate`). Si el estado esta en un backend remoto (S3, GCS, etc.), es responsabilidad del administrador asegurar que el acceso al bucket y los logs de Terraform Cloud esten adecuadamente controlados y cifrados.

## Outputs complejos

Los outputs pueden exponer estructuras de datos completas, no solo valores simples:

```hcl
output "instancias_info" {
  description = "Informacion detallada de todas las instancias"
  value = {
    for idx, instancia in aws_instance.web :
    instancia.tags.Name => {
      id           = instancia.id
      public_ip    = instancia.public_ip
      private_ip   = instancia.private_ip
      availability_zone = instancia.availability_zone
    }
  }
}
```

Este output genera un mapa que puede ser consumido por otros modulos o scripts con informacion estructurada de todas las instancias creadas.

## Precondiciones y postcondiciones

A partir de Terraform 1.2, los bloques `output` soportan `precondition` y `postcondition` para validar que el valor cumple ciertas condiciones antes o despues de ser expuesto:

```hcl
output "instance_public_ip" {
  value = aws_instance.web.public_ip

  postcondition {
    condition     = self.value != ""
    error_message = "La instancia no tiene una IP publica asignada."
  }
}
```

La palabra clave `self.value` dentro del bloque `postcondition` se refiere al valor del output. Si la condicion falla, Terraform detiene la ejecucion y muestra el mensaje de error.

