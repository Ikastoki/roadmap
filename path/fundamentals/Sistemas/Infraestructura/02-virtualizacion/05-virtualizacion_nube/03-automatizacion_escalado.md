# Automatizacion y escalado

La automatizacion y el escalado automatico son las funcionalidades que distinguen a la nube de la virtualizacion local. En un entorno local, escalar requiere comprar hardware, instalarlo y configurarlo. En la nube, escalar es cuestion de una llamada a la API o una configuracion en la consola. Esta capacidad permite que las aplicaciones se adapten automaticamente a la demanda sin intervencion manual.

## Imagenes y plantillas

Las imagenes son VMs preconfiguradas que sirven como punto de partida para crear nuevas instancias. En lugar de instalar el SO y las aplicaciones manualmente en cada nueva instancia, se parte de una imagen que ya incluye todo configurado.

```bash
# AWS: crear imagen (AMI) desde una instancia
aws ec2 create-image \
  --instance-id i-0123456789abcdef0 \
  --name "plantilla-web-server-v2"

# AWS: lanzar instancia desde imagen
aws ec2 run-instances \
  --image-id ami-0123456789abcdef0 \
  --instance-type t3.medium
```

Las imagenes de marketplace son imagenes preconfiguradas por terceros: bases de datos, servidores web, paneles de administracion, todo listo para usar.

## Autoescalado (Auto Scaling)

El autoescalado ajusta automaticamente el numero de instancias segun la demanda. Cuando la carga aumenta, se lanzan nuevas instancias; cuando disminuye, se eliminan las sobrantes.

```bash
# AWS: crear grupo de autoescalado
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name mi-asg \
  --launch-configuration-name mi-lc \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 3 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb"
```

**Politicas de escalado:**

- **Basada en CPU:** escalar cuando el uso de CPU supera un umbral (ej. 70%).
- **Basada en memoria:** escalar cuando la memoria disponible es baja.
- **Basada en trafico:** escalar segun las peticiones por segundo.
- **Programada:** escalar en horarios predefinidos (ej. mas instancias en horario laboral).

```bash
# AWS: politica de escalado por CPU
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name mi-asg \
  --policy-name escalar-por-cpu \
  --scaling-adjustment 1 \
  --adjustment-type ChangeInCapacity \
  --threshold 70
```

## Infraestructura como Codigo (IaC)

IaC permite definir la infraestructura completa en archivos de codigo que se pueden versionar, revisar y ejecutar de forma repetible.

```hcl
# Terraform: definir una VPC y una instancia
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id     = aws_subnet.main.id

  tags = {
    Name = "servidor-web"
  }
}
```

```bash
# Ejecutar Terraform
terraform init
terraform plan
terraform apply
```

```yaml
# CloudFormation (AWS): definir recurso
Resources:
  MiServidorWeb:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.medium
      ImageId: ami-0c55b159cbfafe1f0
      KeyName: mi-clave
```

## Despliegue azul-verde y canario

Estrategias de despliegue que minimizan el riesgo al actualizar aplicaciones:

- **Azul-verde:** desplegar la nueva version en un conjunto de instancias (verde) mientras la version antigua sigue activa (azul). Rediradir el trafico gradualmente.
- **Canario:** desplegar la nueva version a un porcentaje pequeno del trafico (ej. 5%). Si no hay errores, escalar al 100%.

> La automatizacion y el escalado transforman la infraestructura de un recurso estatico a un servicio elastico. Las aplicaciones en la nube pueden crecer y reducirse automaticamente, adaptandose a la demanda en tiempo real sin intervencion manual ni inversion en hardware.

