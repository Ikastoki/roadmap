# Consideraciones de seguridad y costos

La nube ofrece flexibilidad y escalabilidad, pero tambiene introduce riesgos de seguridad y sorpresas en la factura que no existen en la virtualizacion local. Gestionar la seguridad y los costos en la nube requiere disciplina, herramientas adecuadas y politicas claras desde el primer dia.

## Seguridad en la nube

El modelo de responsabilidad compartida define que el proveedor protege la infraestructura y que el usuario protege los datos y configuraciones:

```
Responsabilidad del proveedor:
├── Seguridad fisica del datacenter
├── Seguridad de la red backbone
├── Disponibilidad del hipervisor
└── Certificaciones de cumplimiento

Responsabilidad del usuario:
├── Configuracion de seguridad de la VPC
├── Gestion de identidades y acceso (IAM)
├── Cifrado de datos en reposo y en transito
├── Parches del SO invitado
├── Configuracion de firewalls (grupos de seguridad)
└── Proteccion de credenciales y claves
```

**Gestion de identidades y acceso (IAM):**

```bash
# AWS: crear usuario IAM con permisos minimos
aws iam create-user --user-name developer
aws iam attach-user-policy \
  --user-name developer \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
```

**Grupos de seguridad:**

```bash
# AWS: grupo de seguridad restrictivo
aws ec2 create-security-group \
  --group-name sg-estricto \
  --description "Solo HTTP desde internet" \
  --vpc-id vpc-xxx

# Solo permitir HTTP (puerto 80)
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# NO abrir SSH desde internet (usar Systems Manager o VPN)
```

**Cifrado:**

```bash
# AWS: cifrar volumen EBS
aws ec2 create-volume \
  --size 100 \
  --encrypted \
  --volume-type gp3

# AWS: cifrar bucket S3
aws s3api put-bucket-encryption \
  --bucket mi-bucket \
  --server-side-encryption-configuration \
    '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

## Gestion de costos

Los costos inesperados son el riesgo economico mas comun en la nube. Una instancia olvidada, un bucket S3 publico o un grupo de autoescalado mal configurado pueden generar facturas de miles de euros.

**Herramientas de monitoreo de costos:**

```bash
# AWS: ver gasto actual
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics "BlendedCost"
```

**Alertas de gasto:**

```bash
# AWS: crear alerta de gasto
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{"BudgetName":"alerta-100usd","BudgetLimit":{"Amount":"100","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL","Threshold":100},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"admin@empresa.com"}]}]'
```

**Buenas practicas de costos:**

- **Eliminar instancias ociosas:** apagar VMs de desarrollo fuera de horario laboral.
- **Usar instancias spot:** hasta 90% de descuento para cargas tolerantes a interrupciones.
- **Right-sizing:** ajustar el tipo de instancia al uso real (una instancia `t3.large` con 5% de CPU usa `t3.small` mejor).
- **Reserved Instances:** compromisos a 1 o 3 anos con descuentos significativos.
- **Etiquetado (*tagging*):** etiquetar todos los recursos con proyecto, equipo y entorno para rastrear costos.

```bash
# AWS: etiquetar instancia
aws ec2 create-tags \
  --resources i-0123456789abcdef0 \
  --tags Key=Proyecto,Value=WebApp Key=Equipo,Value=Backend
```

> La nube es poderosa pero no perdonan errores. Una instancia `p4d.24xlarge` (GPU A100) olvidada durante un mes puede costar mas de 20,000 USD. La combinacion de politicas de seguridad restrictivas y monitoreo de costos activo es lo que separa a una implementacion exitosa de un desastre financiero.

