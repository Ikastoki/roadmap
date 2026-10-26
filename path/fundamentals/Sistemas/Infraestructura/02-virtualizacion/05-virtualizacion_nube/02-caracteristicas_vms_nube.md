# Caracteristicas de las VMs en la nube

Las instancias en la nube no son simplemente VMs tradicionales ejecutandose en servidores remotos. Los proveedores de nube añaden capas de abstraccion, servicios gestionados y configuraciones de red que van mucho mas alla de una VM local. Tipos de instancia, almacenamiento efímero vs. persistente, redes virtuales privadas (VPC) y seguridad basada en grupos de seguridad son conceptos que todo usuario de nube debe comprender.

## Tipos de instancia

Cada proveedor ofrece multiples tipos de instancias optimizadas para diferentes cargas de trabajo:

**Instancias de computacion optimizadas:**
- Alto ratio de CPU/RAM.
- Ideales para servidores web, procesamiento en lote, compilacion.
- AWS: `t3`, `c5`, `c6i` | Azure: `F-series` | GCP: `c2`, `c2d`.

**Instancias de memoria optimizada:**
- Alto ratio de RAM/CPU.
- Ideales para bases de datos en memoria, caches, analisis de datos.
- AWS: `r5`, `r6i` | Azure: `E-series` | GCP: `m2`, `n2-highmem`.

**Instancias con GPU:**
- Incluyen GPUs dedicadas para computacion paralela.
- Ideales para machine learning, renderizado, video, ciencia de datos.
- AWS: `p4d`, `g5` | Azure: `NC-series` | GCP: `a2`, `g2`.

**Instancias de almacenamiento optimizado:**
- Acceso rapido a disco NVMe local.
- Ideales para bases de datos transaccionales, data warehousing.
- AWS: `i3`, `d3` | Azure: `Ls-series` | GCP: `b2`.

## Almacenamiento

**Almacenamiento efímero (local):**
- Disco fisico directamente conectado al servidor donde ejecuta la instancia.
- Rapido pero se pierde al apagar la instancia.
- Util para swap, cache, archivos temporales.

**Almacenamiento persistente (bloque):**
- Disco virtual que persiste independientemente del ciclo de vida de la instancia.
- AWS: EBS (Elastic Block Store) | Azure: Managed Disks | GCP: Persistent Disk.
- Se puede adjuntar y desadjuntar de diferentes instancias.

```bash
# AWS: crear volumen EBS y adjuntarlo
aws ec2 create-volume \
  --size 100 \
  --volume-type gp3 \
  --availability-zone eu-west-1a

aws ec2 attach-volume \
  --volume-id vol-0123456789abcdef0 \
  --instance-id i-0123456789abcdef0 \
  --device /dev/sdf
```

**Almacenamiento de objetos:**
- Para archivos estaticos, imagenes, backups, datos de aplicacion.
- AWS: S3 | Azure: Blob Storage | GCP: Cloud Storage.
- Acceso via HTTP/HTTPS, no se monta como disco.

## Redes virtuales (VPC)

Cada instancia se ejecuta dentro de una VPC (Virtual Private Cloud), una red virtual aislada que el usuario configura.

```bash
# AWS: crear VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Crear subred publica
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone eu-west-1a

# Crear grupo de seguridad
aws ec2 create-security-group \
  --group-name mi-sg \
  --description "Permitir SSH y HTTP" \
  --vpc-id vpc-0123456789abcdef0

# Reglas de entrada
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

## Balanceadores de carga

Distribuyen el trafico entre multiples instancias para mejorar disponibilidad y rendimiento.

```bash
# AWS: crear Application Load Balancer
aws elbv2 create-load-balancer \
  --name mi-alb \
  --subnets subnet-aaa subnet-bbb \
  --security-groups sg-0123456789abcdef0 \
  --type application
```

> Las VMs en la nube son el punto de partida de la infraestructura moderna. Comprender los tipos de instancia, el almacenamiento y las redes virtuales es fundamental para construir aplicaciones escalables, seguras y经济icas en la nube.

