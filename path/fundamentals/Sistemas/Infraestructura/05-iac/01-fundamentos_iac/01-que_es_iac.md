# Que es Infraestructura como Codigo

La infraestructura como codigo (IaC) representa un cambio de paradigma en la gestion de recursos tecnologicos: en lugar de configurar servidores, redes y servicios mediante procesos manuales o scripts ad-hoc, se describe el estado deseado de la infraestructura en archivos de configuracion legibles por maquina que pueden versionarse, probarse y ejecutarse de forma automatica. Este enfoque convierte la infraestructura en un artifacto de software mas, sujeto a las mismas practicas de ingenieria que el codigo de aplicacion.

El problema que resuelve IaC es fundamental: en entornos gestionados manualmente, cada servidor se convierte en un "snowflake" unico e irrepetible, donde pequeñas diferencias en la configuracion se acumulan hasta crear inconsistencias criticas. Este fenomeno, conocido como *configuration drift*, provoca que entornos que deberian ser identicos (desarrollo, staging, produccion) se comporten de manera diferente, generando bugs dificiles de diagnosticar y fallos en despliegues que funcionan en un entorno pero no en otro.

## Principios fundamentales

IaC se sustenta en cinco principios que transforman la gestion de infraestructura:

**Versionable**: Los archivos de configuracion se almacenan en repositorios Git, permitiendo historial completo de cambios, comparaciones entre versiones y la capacidad de volver a un estado anterior. Cada modificacion en la infraestructura queda registrada con autor, fecha y justificacion.

**Reusable**: Los componentes de infraestructura se componen como modulos que pueden combinarse para crear entornos completos. Un modulo de red virtual, uno de base de datos y uno de balanceador de carga pueden reutilizarse en multiples proyectos con configuraciones diferentes.

**Consistente**: La ejecucion del mismo codigo produce exactamente la misma infraestructura, eliminando las variaciones humanas que introducen configuracion inconsistente. No importa si ejecutas la configuracion diez veces: el resultado sera identico.

**Idempotente**: Ejecutar la configuracion multiples veces produce el mismo resultado que ejecutarla una sola vez. Si el recurso ya existe en el estado deseado, la herramienta no lo modifica. Este principio es critico porque permite ejecuciones seguras sin efectos colaterales no deseados.

**Auditable**: Cada cambio queda registrado en el historial de versiones, creando una trazabilidad completa de quien cambio que, cuando y por que. Esto cumple con requisitos de compliance y facilita la resolucion de incidentes.

## Comparacion: enfoque manual vs IaC

### Enfoque manual (antes)

```bash
# Crear una VPC en AWS manualmente con AWS CLI
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Crear subnets en diferentes zonas de disponibilidad
aws ec2 create-subnet --vpc-id vpc-abc123 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-abc123 --cidr-block 10.0.2.0/24 --availability-zone us-east-1b

# Configurar tablas de ruteo
aws ec2 create-route-table --vpc-id vpc-abc123
aws ec2 associate-route-table --route-table-id rtb-def456 --subnet-id subnet-ghi789

# Crear grupo de seguridad
aws ec2 create-security-group --group-name web-sg --description "Web traffic" --vpc-id vpc-abc123
aws ec2 authorize-security-group-ingress --group-id sg-jkl012 --protocol tcp --port 443 --cidr 0.0.0.0/0

# Lanzar instancia EC2
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --subnet-id subnet-ghi789 \
  --security-group-ids sg-jkl012 \
  --key-name my-key
```

> Este enfoque tiene multiples problemas: no hay registro de los valores usados, los IDs de recursos deben guardarse manualmente, y repetir el proceso genera recursos duplicados sin control.

### Enfoque con Terraform (despues)

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_subnet" "public_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
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
}

resource "aws_instance" "web" {
  ami                    = "ami-0abcdef1234567890"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public_a.id
  vpc_security_group_ids = [aws_security_group.web.id]
}
```

> La diferencia es inmediata: la configuracion es declarativa, versionable y autoexplicativa. Las dependencias entre recursos se resuelven automaticamente y el estado se gestiona internamente por la herramienta.

## Eliminacion del configuration drift

El *configuration drift* ocurre cuando el estado real de la infraestructura se desvía del estado esperado. En entornos manuales, este desvío es gradual e imperceptible: un ingeniero modifica un parametro "porque lo necesitaba urgente", otro ajusta una regla de firewall "solo para probar", y en cuestion de semanas el entorno de produccion ya no se parece al que se configuro inicialmente.

IaC resuelve esto mediante el concepto de *state file*. Terraform, por ejemplo, almacena en un archivo JSON el mapeo entre los recursos definidos en el codigo y los IDs reales en el proveedor de nube. Antes de cada plan o aplicacion, Terraform compara el estado deseado (codigo) con el estado actual (state file + recursos reales) y reporta exactamente que cambiaria. Esto permite detectar y corregir drift automaticamente.

```bash
# Ver el plan de cambios antes de aplicar
terraform plan

# Salida típica mostrando drift detectado
# aws_security_group.web will be updated in-place
#   ~ ingress {
#       ~ from_port = 80 -> 443
#     }
```

