# Comandos de gestion de estado

Terraform proporciona un conjunto de subcomandos dentro de `terraform state` para inspeccionar, modificar y manipular directamente el archivo de estado. Estos comandos son herramientas de administracion avanzada: permiten corregir desviaciones, eliminar recursos del control de Terraform o importar infraestructura existente sin necesidad de editar manualmente el JSON del estado.

## terraform state list

Enumera todos los recursos registrados en el estado actual. La salida muestra cada recurso con su tipo y nombre local en el formato `tipo.nombre`:

```bash
terraform state list

data.aws_availability_zones.available
aws_vpc.main
aws_subnet.public
aws_subnet.private
aws_instance.web
aws_security_group.web_sg
```

El comando admite filtros para limitar la salida a recursos de un tipo especifico o bajo un modulo determinado:

```bash
# Recursos de un tipo concreto
terraform state list aws_instance

# Recursos dentro de un modulo
terraform state list module.vpc
```

Esta salida es util para obtener una vision rapida de todo lo que Terraform gestiona, especialmente en proyectos con decenas o cientos de recursos distribuidos en multiples modulos.

## terraform state show

Muestra los atributos completos de un recurso especifico almacenados en el estado. A diferencia de `terraform state list`, que solo enumera identificadores, `show` revela todos los valores, incluyendo aquellos que no se exponen como outputs ni son faciles de consultar via consola cloud.

```bash
terraform state show aws_instance.web

# aws_instance.web:
resource "aws_instance" "web" {
    ami                          = "ami-0c55b159cbfafe1f0"
    arn                          = "arn:aws:ec2:us-east-1:123456789012:instance/i-0abcd1234efgh5678"
    associate_public_ip_address  = true
    availability_zone            = "us-east-1a"
    id                           = "i-0abcd1234efgh5678"
    instance_state               = "running"
    instance_type                = "t2.micro"
    private_ip                   = "10.0.1.25"
    public_ip                    = "54.123.45.67"
    subnet_id                    = "subnet-0fedcba9876543210"
    vpc_security_group_ids       = ["sg-0123456789abcdef0"]
    tags                         = {
        "Name" = "web-server-prod"
    }
}
```

La informacion devuelta por `show` proviene directamente del estado, no de una llamada a la API del proveedor. Esto significa que el comando funciona incluso sin credenciales configuradas, siempre que el archivo de estado sea accesible. La desventaja es que los valores pueden estar desactualizados si el estado no refleja la ultima realidad.

## terraform state rm

Elimina un recurso del estado sin destruir el recurso real en el proveedor cloud. Es una operacion delicada: una vez eliminado, Terraform pierde todo conocimiento de ese recurso. El recurso fisico continuara existiendo y generando costos, pero Terraform no lo modificara ni lo eliminara.

```bash
terraform state rm aws_instance.web

Removed aws_instance.web from the state.
```

Casos de uso tipicos:

- **Migrar un recurso a otro state file**: extraer un recurso del estado actual y despues importarlo en otro estado.
- **Dejar de gestionar un recurso**: cuando un recurso debe pasar a gestionarse manualmente o con otra herramienta.
- **Recrear un recurso corrupto**: eliminar del estado para que el proximo `apply` lo cree de nuevo, sin tener que destruir manualmente el recurso real.

`state rm` tambien acepta el flag `-dry-run` para revisar que recursos se eliminarian sin ejecutar la operacion:

```bash
terraform state rm -dry-run aws_instance.web

Would remove aws_instance.web from the state.
```

> Es recomendable usar `terraform plan` despues de un `state rm` para confirmar que Terraform intentara crear un nuevo recurso del mismo tipo, lo cual suele ser el objetivo deseado.

## terraform import

Importa un recurso existente del proveedor cloud al estado de Terraform, permitiendo que Terraform empiece a gestionarlo sin necesidad de crearlo desde cero. Es la operacion inversa de `state rm`.

```bash
terraform import aws_instance.web i-0abcd1234efgh5678

aws_instance.web: Importing from ID "i-0abcd1234efgh5678"...
aws_instance.web: Import prepared!
  Prepared aws_instance for import
aws_instance.web: Refreshing state... [id=i-0abcd1234efgh5678]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

La sintaxis es `terraform import <resource_type>.<resource_name> <ID_del_proveedor>`. El recurso debe estar definido previamente en el codigo HCL con los atributos minimos que Terraform necesita para identificarlo. Tras la importacion, es necesario ejecutar `terraform plan` para detectar diferencias entre el estado real del recurso y la configuracion declarada en el codigo, y ajustar esta ultima para que coincida.

Requisitos para una importacion exitosa:

1. El bloque `resource` debe existir en el codigo HCL (aunque este vacio en sus atributos).
2. El ID del recurso en el proveedor debe ser conocido (AMI ID, nombre de bucket, ARN, etc.).
3. Algunos recursos requieren atributos obligatorios en el bloque `resource` antes de la importacion.

```hcl
# main.tf: bloque minimo necesario antes de importar
resource "aws_instance" "web" {
  # Se rellenara tras importar
}
```

> No todos los recursos soportan importacion. La documentacion de cada recurso en el Terraform Registry indica si es importable y cual es el formato del ID requerido.

## Consideraciones de seguridad en comandos de estado

Tanto `terraform state show` como `terraform state list` pueden exponer informacion sensible almacenada en el estado. En entornos donde el acceso al archivo de estado debe restringirse, estos comandos heredan las mismas politicas de control de acceso del backend:

- En S3, las politicas IAM del bucket y la tabla DynamoDB determinan quien puede ejecutar estos comandos.
- En Terraform Cloud, los permisos se gestionan a nivel de equipo y workspace.
- Un usuario con acceso de solo lectura al estado puede inspeccionar todos los atributos de los recursos gestionados, incluyendo aquellos marcados como `sensitive`.

