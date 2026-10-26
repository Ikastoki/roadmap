# La nube como evolucion de la virtualizacion

La computacion en nube es la evolucion natural de la virtualizacion local. Donde la virtualizacion tradicional requiere comprar, instalar y mantener servidores fisicos para ejecutar VMs, la nube lleva esta capa de abstraccion a un nivel superior: proveedores como AWS, Azure y Google Cloud ejecutan las VMs en su infraestructura y las ofrecen bajo demanda a traves de internet. El usuario ya no piensa en servidores, hipervisores ni almacenamiento; solo solicita recursos y paga por lo que usa.

Este modelo transformo la forma en que las empresas consumen infraestructura de TI: de una inversion inicial fija (CAPEX) a un gasto operativo variable (OPEX).

## IaaS: Infraestructura como Servicio

IaaS (Infrastructure as a Service) es el modelo de nube mas cercano a la virtualizacion tradicional. El proveedor ofrece VMs (llamadas *instancias*) listas para usar, con CPU, memoria, almacenamiento y red virtuales.

```bash
# AWS EC2: crear una instancia con AWS CLI
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name mi-clave-ssh \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0
```

```bash
# Azure: crear una VM
az vm create \
  --name mi-servidor \
  --resource-group mi-grupo \
  --image UbuntuLTS \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys
```

```bash
# Google Cloud: crear una instancia
gcloud compute instances create mi-servidor \
  --zone=europe-west1-b \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud
```

## Abstraccion total del hardware

En la nube, el hardware fisico es completamente invisible. No sabes en que servidor fisico esta ejecutandose tu instancia, que CPU tiene exactamente ni que discos estan conectados. Todo esta abstraido tras las APIs del proveedor.

Esta abstraccion tiene ventajas:
- **No hay mantenimiento de hardware:** el proveedor se encarga.
- **No hay compras de hardware:** escalas bajo demanda.
- **No hay preocupaciones de failover:** el proveedor gestiona la disponibilidad.

Y desventajas:
- **Menor control:** no puedes elegir el hardware exacto.
- **Dependencia del proveedor:** migrar de AWS a Azure no es trivial.
- **Costos variables:** si no monitoreas, la factura puede sorprenderte.

## Modelo de pago por uso

En virtualizacion local, compras un servidor de 64 GB de RAM por 5000 euros, aunque solo necesites 10 GB la mayoria del tiempo. En la nube, pagas por la hora (o segundo) que la instancia esta encendida y por los recursos que consume.

```
Comparacion de costos (ejemplo conceptual):
Virtualizacion local:
  - Servidor: 5000 EUR (inversion inicial)
  - Energia: 50 EUR/mes
  - Mantenimiento: 100 EUR/mes
  - Total anual: 7300 EUR (incluso si el servidor esta ocioso)

Nube:
  - Instancia t3.medium: 0.0416 USD/hora
  - Uso 8 horas/dia, 22 dias/mes:
  - Total mensual: ~73 USD (~67 EUR)
  - Si no la usas, no pagas
```

## Elasticidad

La nube permite escalar recursos automaticamente segun la demanda. Un servidor local tiene una capacidad fija; una instancia en la nube puede crecer o reducirse en minutos.

```bash
# AWS: escalar instancia a tipo mayor
aws ec2 modify-instance-attribute \
  --instance-id i-0123456789abcdef0 \
  --instance-type "{\"Value\": \"t3.large\"}"
```

> La nube no reemplaza la virtualizacion: la lleva a su maxima expresion. La virtualizacion local resuelve el problema de eficiencia del hardware; la nube resuelve el problema de acceso a infraestructura sin inversion inicial. Juntas, representan la evolucion completa de como consumimos poder de calculo.

