# Gestion de salidas

Despues de ejecutar un despliegue, la infraestructura creada tiene datos utiles que no forman parte de la definicion pero que son necesarios para otros procesos: la direccion IP publica de un balanceador de carga, el identificador de un rol IAM, el ARN de un topic de SNS o la cadena de conexion a una base de datos. Las salidas (outputs) son el mecanismo formal para exponer estos valores de forma estructurada, tanto para consumo humano como para integracion automatizada entre herramientas o modulos.

El concepto de salida resuelve un problema fundamental de modularidad: si un modulo de Terraform crea una VPC y sus subnets, los modulos que consumen esa VPC necesitan conocer el CIDR y los IDs de las subnets sin tener acceso directo al estado del modulo fuente. Las salidas actuan como una interfaz publica del modulo, encapsulando los datos relevantes y ocultando los detalles de implementacion. Este patron es analogo al concepto de interfaz en programacion orientada a objetos: solo se expone lo necesario, manteniendo el resto como implementacion interna.

## Definicion de salidas en Terraform

```hcl
output "ip_balanceador" {
  description = "Direccion IP publica del balanceador de carga"
  value       = aws_lb.api.dns_name
}

output "endpoint_api" {
  description = "URL completa del endpoint de la API"
  value       = "https://${aws_lb.api.dns_name}"
}

output "ids_subnets_privadas" {
  description = "Lista de IDs de las subnets privadas creadas"
  value       = aws_subnet.privada[*].id
}
```

Las salidas se definen con un nombre unico dentro del modulo, una descripcion y un valor que puede ser cualquier expresion de Terraform: un atributo de recurso, una interpolacion, una referencia a variable o incluso una expresion condicional. Cuando se ejecuta `terraform output`, todos los valores se muestran en la consola. Cuando un modulo exporta salidas, otros modulos las consumen con el prefijo del nombre del modulo.

```hcl
module "red" {
  source = "./modules/red"
}

module "api" {
  source = "./modules/api"
  subnet_ids = module.red.ids_subnets_privadas
}
```

## Marcado como sensible

Algunos datos producidos por un despliegue no deben exponerse en la consola ni en los logs de ejecucion: contrasenas generadas, certificados, tokens de acceso o cadenas de conexion con credenciales. Terraform ofrece el atributo `sensitive` para evitar que estos valores se muestren en la salida estandar, en los planes de ejecucion o en el estado sin cifrar.

```hcl
output "password_base_datos" {
  description = "Contrasena generada para la instancia RDS"
  value       = aws_db_instance_principal.password
  sensitive   = true
}

output "conexion_base_datos" {
  description = "Cadena de conexion completa"
  value       = "postgresql://${aws_db_instance_principal.username}:${aws_db_instance_principal.password}@${aws_db_instance_principal.endpoint}"
  sensitive   = true
}
```

> Cuando una salida se marca como `sensitive`, Terraform la oculta en la consola mostrando `(sensitive value)`. Sin embargo, el valor permanece en texto plano dentro del archivo de estado (`terraform.tfstate`). El marcado sensible no sustituye al cifrado del estado; es una capa de proteccion adicional para evitar la exposicion accidental en interacciones humanas con la herramienta.

**Consumo de salidas sensibles desde otros modulos**

```hcl
output "url_conexion" {
  value     = module.base_datos.conexion_base_datos
  sensitive = true
}

resource "kubernetes_secret" "db_credentials" {
  metadata {
    name = "db-credentials"
  }

  data = {
    connection_string = module.base_datos.conexion_base_datos
  }
}
```

Cuando un modulo consume una salida sensible de otro modulo, Terraform mantiene la propiedad sensible en la cadena de dependencias. Esto impide que el valor se muestre en el plan de ejecucion del modulo consumidor, incluso si el modulo consumidor no lo marca explicitamente como sensible.

## Salidas en CloudFormation

CloudFormation no tiene un bloque de salidas equivalente en estructura, pero si un objeto `Outputs` en la seccion inferior del template que permite exportar valores con fines de referencia entre stacks.

```yaml
Outputs:
  BalanceadorDnsName:
    Description: DNS del load balancer
    Value: !GetAtt Balanceador.DNSName
    Export:
      Name: !Sub "${AWS::StackName}-LB-DNS"

  EndpointApi:
    Description: URL del endpoint
    Value: !Sub "https://${Balanceador.DNSName}"
```

Las exportaciones de CloudFormation se referencian desde otros stacks con `Fn::ImportValue` o `!ImportValue`. Este mecanismo permite construir cadenas de dependencias entre stacks, donde el output de un stack se convierte en el input de otro sin necesidad de passar valores manualmente.

```yaml
# Stack de red (referencia)
Resources:
  EndpointApi:
    Type: AWS::CloudFormation::WaitConditionHandle
    Metadata:
      Ref: BalanceadorDnsName

# Stack de aplicacion (consumo)
Parameters:
  EndpointApi:
    Type: String
    Default: !ImportValue miapp-network-LB-DNS
```

## Salidas en Pulumi

Pulumi exporta valores desde un programa TypeScript o Python usando `pulumi.export()`. Estos valores aparecen en la consola despues de la ejecucion y estan disponibles en la interfaz web de Pulumi.

```typescript
import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

const lb = new aws.elasticloadbalancingv2.LoadBalance("api", { /* ... */ });

pulumi.export("loadBalancerDns", lb.dnsName);
pulumi.export("vpcId", vpc.id);
```

Las salidas de Pulumi se almacenan en el backend del servicio (Pulumi Cloud o un backend self-hosted) y se recuperan con `pulumi stack output`. Para marcar un valor como sensible, se envuelve con `pulumi.secret()`:

```typescript
pulumi.export("dbPassword", pulumi.secret(dbInstance.password));
```

> Las salidas no son meros datos de conveniencia: son contratos de interfaz entre modulos, stacks y herramientas. Un output mal documentado o con un nombre ambiguo genera confusion en equipos grandes y dificulta la composicion de infraestructura. Las descripciones de cada output deben ser tan explicitas como las documentacion de una API publica.

