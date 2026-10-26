# Enfoques Imperativo y Declarativo en IaC

La distincion entre imperativo y declarativo es el eje central sobre el que se estructuran todas las herramientas de infraestructura como codigo. Comprender esta diferencia no es meramente teorico: determina la forma en que interactuas con la infraestructura, como se gestionan los errores, y que tan predecible es el resultado de cada ejecucion. Ambos enfoques resuelven el mismo problema (automatizar infraestructura), pero lo hacen desde filosofias radicalmente opuestas.

En terminos simples, un enfoque imperativo te dice *como* llegar al destino paso a paso, mientras que el declarativo describe *donde* quieres llegar y deja que la herramienta calcule la ruta. Esta diferencia tiene implicaciones practicas profundas: el enfoque imperativo requiere que conozcas cada operacion de la API del proveedor, mientras que el declarativo te abstrae de la implementacion y te permite enfocarte en el resultado.

## Enfoque imperativo

El modelo imperativo ejecuta una secuencia explicita de comandos que transforman el estado actual de la infraestructura hasta alcanzar el estado deseado. Cada paso es una operacion concreta sobre un recurso especifico, y el orden de ejecucion es determinista y controlado por el usuario.

La caracteristica definitoria del imperativo es que *tu* defines el flujo de control: condicionales, bucles, manejo de errores y llamadas a APIs. Esto ofrece control total pero introduce complejidad significativa, especialmente cuando se trata de manejar estados parciales o recursos que ya existen.

### Script bash con AWS CLI

```bash
#!/bin/bash
set -euo pipefail

VPC_CIDR="10.0.0.0/16"
SUBNET_CIDR_A="10.0.1.0/24"
SUBNET_CIDR_B="10.0.2.0/24"

# Paso 1: Crear VPC
echo "Creando VPC..."
VPC_ID=$(aws ec2 create-vpc --cidr-block "$VPC_CIDR" \
  --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames
echo "VPC creada: $VPC_ID"

# Paso 2: Crear subnets
echo "Creando subnet A..."
SUBNET_A=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "$SUBNET_CIDR_A" \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)

echo "Creando subnet B..."
SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "$SUBNET_CIDR_B" \
  --availability-zone us-east-1b \
  --query 'Subnet.SubnetId' --output text)

# Paso 3: Crear Internet Gateway
echo "Creando Internet Gateway..."
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"

# Paso 4: Crear tabla de ruteo y asociar
RT_ID=$(aws ec2 create-route-table --vpc-id "$VPC_ID" \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id "$RT_ID" \
  --destination-cidr-block "0.0.0.0/0" --gateway-id "$IGW_ID"
aws ec2 associate-route-table --route-table-id "$RT_ID" --subnet-id "$SUBNET_A"
aws ec2 associate-route-table --route-table-id "$RT_ID" --subnet-id "$SUBNET_B"

# Paso 5: Crear grupo de seguridad
SG_ID=$(aws ec2 create-security-group \
  --group-name "web-sg" \
  --description "Web traffic" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port 443 --cidr 0.0.0.0/0

# Paso 6: Lanzar instancia
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --subnet-id "$SUBNET_A" \
  --security-group-ids "$SG_ID" \
  --key-name my-key

echo "Infraestructura desplegada exitosamente"
```

### Python con boto3

```python
import boto3

ec2 = boto3.resource('ec2', region_name='us-east-1')
client = boto3.client('ec2', region_name='us-east-1')

# Crear VPC
vpc = ec2.create_vpc(CidrBlock='10.0.0.0/16')
vpc.modify_attribute(EnableDnsSupport={'Value': True})
vpc.modify_attribute(EnableDnsHostnames={'Value': True})

# Crear subnets
subnet_a = ec2.create_subnet(
    VpcId=vpc.id,
    CidrBlock='10.0.1.0/24',
    AvailabilityZone='us-east-1a'
)
subnet_b = ec2.create_subnet(
    VpcId=vpc.id,
    CidrBlock='10.0.2.0/24',
    AvailabilityZone='us-east-1b'
)

# Crear Internet Gateway
igw = ec2.create_internet_gateway()
igw.attach_to_gateway(VpcId=vpc.id)

# Crear tabla de ruteo con ruta por defecto
rt = ec2.create_route_table(VpcId=vpc.id)
rt.create_route(DestinationCidrBlock='0.0.0.0/0', GatewayId=igw.id)
rt.associate_with_subnet(SubnetId=subnet_a.id)
rt.associate_with_subnet(SubnetId=subnet_b.id)

# Crear grupo de seguridad
sg = ec2.create_security_group(
    GroupName='web-sg',
    Description='Web traffic',
    VpcId=vpc.id
)
sg.authorize_ingress(
    IpPermissions=[{
        'IpProtocol': 'tcp',
        'FromPort': 443,
        'ToPort': 443,
        'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
    }]
)

# Lanzar instancia
ec2.create_instances(
    ImageId='ami-0abcdef1234567890',
    InstanceType='t3.micro',
    SubnetId=subnet_a.id,
    SecurityGroupIds=[sg.id],
    KeyName='my-key'
)
```

> Los scripts imperativos son secuenciales y deterministas. Sin embargo, carecen de mecanismos nativos para determinar si un recurso ya existe o necesita ser creado, lo que obliga a implementar logica de verificacion manual.

## Enfoque declarativo

El modelo declarativo describe el *estado final* deseado de la infraestructura. El motor de la herramienta (Terraform, CloudFormation, Pulumi) analiza la definicion, compara con el estado actual calculado desde los recursos reales, y genera un plan de ejecucion que contiene las operaciones minimas necesarias para alcanzar el estado deseado.

La diferencia fundamental es que el usuario no especifica operaciones CRUD, sino propiedades de los recursos. No se crea, actualiza ni elimina nada directamente; se declara como deberia ser y la herramienta determina que accion es necesaria. Esto habilita la idempotencia porque re-ejecutar la misma declaracion produce el mismo resultado sin importar el estado actual.

### Terraform

```hcl
# main.tf

provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "vpc-produccion"
  }
}

resource "aws_subnet" "public_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "subnet-public-a"
  }
}

resource "aws_subnet" "public_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"

  tags = {
    Name = "subnet-public-b"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "igw-produccion"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "rt-public"
  }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "sg-web"
  }
}

resource "aws_instance" "web" {
  ami                    = "ami-0abcdef1234567890"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public_a.id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = "my-key"

  tags = {
    Name = "web-server"
  }
}
```

### Kubernetes manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### AWS CloudFormation

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Infraestructura web completa'

Parameters:
  EnvironmentName:
    Type: String
    Default: produccion

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web traffic
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
```

### Pulumi (TypeScript)

```typescript
import * as aws from "@pulumi/aws";

const vpc = new aws.ec2.Vpc("main", {
    cidrBlock: "10.0.0.0/16",
    enableDnsSupport: true,
    enableDnsHostnames: true,
    tags: { Name: "vpc-produccion" },
});

const subnetA = new aws.ec2.Subnet("public-a", {
    vpcId: vpc.id,
    cidrBlock: "10.0.1.0/24",
    availabilityZone: "us-east-1a",
    mapPublicIpOnLaunch: true,
    tags: { Name: "subnet-public-a" },
});

const webSg = new aws.ec2.SecurityGroup("web-sg", {
    description: "Web traffic",
    vpcId: vpc.id,
    ingress: [{
        fromPort: 443,
        toPort: 443,
        protocol: "tcp",
        cidrBlocks: ["0.0.0.0/0"],
    }],
    egress: [{
        fromPort: 0,
        toPort: 0,
        protocol: "-1",
        cidrBlocks: ["0.0.0.0/0"],
    }],
});

const webInstance = new aws.ec2.Instance("web", {
    ami: "ami-0abcdef1234567890",
    instanceType: "t3.micro",
    subnetId: subnetA.id,
    vpcSecurityGroupIds: [webSg.id],
    keyName: "my-key",
    tags: { Name: "web-server" },
});
```

## Comparativa directa

| Aspecto | Imperativo | Declarativo |
|---------|-----------|-------------|
| Definicion | Secuencia de operaciones | Estado deseado |
| Orden de ejecucion | Controlado por el usuario | Determinado por la herramienta |
| Idempotencia | Requiere implementacion manual | Nativa por diseno |
| Manejo de errores | Responsabilidad del script | Gestionado por el motor |
| Deteccion de drift | No existe | Automatica |
| Curva de aprendizaje | Depende del conocimiento de APIs | Depende del lenguaje de declaracion |
| Control fino | Total | Limitado a lo que expone la herramienta |

> En la practica moderna de IaC, el enfoque declarativo es el estandar de facto. Las herramientas imperativas siguen siendo utiles para tareas puntuales o scripting de utilidad, pero la infraestructura de produccion se gestiona casi exclusivamente con herramientas declarativas.

