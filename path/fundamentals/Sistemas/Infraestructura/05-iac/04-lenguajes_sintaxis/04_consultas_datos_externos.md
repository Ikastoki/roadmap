# Consultas de datos externos

La definicion de infraestructura en IaC no existe en un vacio: las decisiones de despliegue dependen de informacion que solo se obtiene en tiempo de ejecucion. Cual es la AMI mas reciente de Amazon Linux en la region de despliegue? Cuantas zonas de disponibilidad estan disponibles en `eu-west-1`? Cual es la politica de IP de un VPC existente que no fue creado con Terraform? Estas preguntas tienen respuestas dinamicas que cambian con el tiempo y no pueden codificarse como valores estaticos en un manifiesto.

Los data sources (fuentes de datos) de Terraform, los dynamic providers de Pulumi y los custom resources de CloudFormation resuelven este problema consultando APIs externas durante la fase de planificacion o ejecucion. A diferencia de los recursos, que representan infraestructura que se gestiona (crear, actualizar, eliminar), las fuentes de datos son de solo lectura: obtienen informacion del entorno sin modificarlo. Este patron permite construir definiciones de infraestructura adaptativas que se ajustan automaticamente a las condiciones del momento.

## Data sources en Terraform

Un data source en Terraform se declara con el bloque `data` y sigue la misma sintaxis que un recurso, pero su ciclo de vida es inverso: no crea nada, solo consulta y devuelve atributos que se consumen en otros bloques.

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

data "aws_availability_zones" "disponibles" {
  state = "available"
}

data "aws_vpc" "existente" {
  tags = {
    Name = "vpc-produccion"
  }
}

data "aws_subnet_ids" "privadas" {
  vpc_id = data.aws_vpc.existente.id
  tags = {
    Tier = "private"
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"
  subnet_id     = data.aws_subnet_ids.privadas.ids[0]

  tags = {
    Name = "app-server"
  }
}
```

La consulta de la AMI mas reciente se ejecuta cada vez que se ejecuta `terraform plan`. Si Amazon publica una nueva version de la AMI, el plan mostrara un cambio en el recurso `aws_instance.app` porque su `ami` referencia un valor que ha cambiado. Este comportamiento es intencional: garantiza que las instancias nuevas siempre arranquen con la ultima version disponible.

> Los data sources no gestionan infraestructura. Si se eliminan del manifiesto, Terraform no intentara eliminar nada en la nube. Sin embargo, si un recurso depende de un data source que falla (por ejemplo, porque no existe ningun recurso que coincida con los filtros), toda la operacion de plan o apply fallara.

**Filtrado y seleccion de datos**

```hcl
data "aws_iam_policy_document" "acceso_s3" {
  statement {
    sid    = "ListBucket"
    effect = "Allow"
    actions = [
      "s3:ListBucket",
    ]
    resources = [
      aws_s3_bucket.datos.arn,
    ]
  }

  statement {
    sid    = "GetObject"
    effect = "Allow"
    actions = [
      "s3:GetObject",
    ]
    resources = [
      "${aws_s3_bucket.datos.arn}/*",
    ]
  }
}
```

El data source `aws_iam_policy_document` construye un documento de politica IAM en formato JSON a partir de bloques declarativos. No consulta una API externa, sino que transforma una definicion estructurada en el formato que espera AWS. Es un data source utilitario, no una consulta en tiempo de ejecucion, pero demuestra como los data sources pueden generar datos derivados que se consumen en recursos posteriores.

## Dynamic providers en Pulumi

Pulumi resuelve el problema de las fuentes de datos externos mediante dynamic providers, que encapsulan logica personalizada en una clase que implementa los metodos `create`, `update` y `delete`. Un dynamic provider puede consultar cualquier API externa durante la creacion de un recurso, y el resultado se almacena en el estado de Pulumi.

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as http from "http";

const latestAmi = new pulumi.dynamic.ResourceProvider("AmiProvider", {
  async create(inputs: { os: string; region: string }) {
    const url = `https://check.example.com/api/ami?os=${inputs.os}&region=${inputs.region}`;
    const data = await fetch(url);
    const json = await data.json();
    return {
      id: json.amiId,
      outs: { amiId: json.amiId, name: json.name },
    };
  },
});
```

Los dynamic providers son la forma mas flexible de integrar datos externos en Pulumi, pero requieren implementar la logica completa de ciclo de vida. Para consultas simples que no requieren gestion de estado, Pulumi tambien ofrece `pulumi.runtime.invoke()`, que ejecuta funciones existentes de un paquete de providers sin crear un recurso gestionado.

## Custom resources en CloudFormation

CloudFormation no tiene un concepto equivalente a los data sources de Terraform. En su lugar, se utilizan custom resources implementadas como lambdas que se ejecutan durante la creacion, actualizacion o eliminacion del stack. Una custom resource puede consultar cualquier API, devolver datos y establecer atributos que se consumen en el template.

```yaml
Parameters:
  Entorno:
    Type: String
    AllowedValues:
      - dev
      - staging
      - produccion

Resources:
  ObtenerAMI:
    Type: Custom::ObtenerAMI
    Properties:
      ServiceToken: !Sub "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:ami-resolver"
      OS: "amazon-linux-2"
      Region: !Ref "AWS::Region"

  InstanciaApp:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !GetAtt ObtenerAMI.AmiId
      InstanceType: t3.medium
```

La lambda `ami-resolver` recibe los parametros del custom resource, consulta la API de EC2 para obtener la AMI mas reciente y devuelve el identificador como atributo del recurso custom. CloudFormation espera a que la lambda complete la ejecucion antes de continuar con la creacion de la instancia.

> Las custom resources en CloudFormation introducen complejidad operativa considerable: requieren una lambda separada, un rol IAM, manejo de errores y retroalimentacion asincrona (usando `cfn-response`). Para consultas simples, a menudo es mas practico utilizar un paso de pre-procesamiento en el pipeline de CI/CD que resuelva los valores dinamicos antes de generar el template.

**Implementacion de la lambda de resolucion**

```python
import boto3
import json
import cfnresponse

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    os_type = event['ResourceProperties']['OS']
    region = event['ResourceProperties']['Region']

    response = ec2.describe_images(
        Owners=['amazon'],
        Filters=[
            {'Name': 'name', 'Values': [f'*{os_type}*']},
            {'Name': 'state', 'Values': ['available']},
        ]
    )

    images = sorted(response['Images'], key=lambda x: x['CreationDate'], reverse=True)
    latest = images[0]

    cfnresponse.send(event, context, cfnresponse.SUCCESS, {
        'AmiId': latest['ImageId'],
        'Name': latest['Name'],
    })
```

La funcion `cfnresponse` envia la respuesta a CloudFormation de forma asincrona. El tipo `cfnresponse.SUCCESS` indica que la operacion se completo correctamente y los atributos proporcionados estan disponibles para `!GetAtt` en el template.

> La consulta de datos externos en IaC plantea un dilema entre determinismo y actualidad. Un data source que consulta la AMI mas reciente en cada ejecucion genera cambios no intencionados en la infraestructura. Terraform resuelve parcialmente este problema con el bloque `lifecycle { ignore_changes = [...] }`, que permite ignorar cambios especificos en un atributo. La decision de cuando fijar un valor y cuando consultarlo dinamicamente es una de las mas importantes en el diseño de una arquitectura de IaC madura.

