# Servicios gestionados en cloud

Minikube y Kind resuelven el problema del desarrollo y testing locales, pero el destino final de cualquier aplicacion Kubernetes es un cluster de produccion. Montar y mantener un cluster Kubernetes desde cero requiere dominar kubeadm, configurar etcd en alta disponibilidad, gestionar certificados TLS, configurar la red de pods, y establecer actualizaciones del plano de control sin downtime. Los servicios gestionados en cloud eliminan esta carga operativa: el proveedor se encarga del plano de control, de la actualizacion de versiones, del parcheo de seguridad, y de la integracion con el ecosistema de servicios cloud (balanceadores, almacenamiento, DNS, monitoring). EKS, GKE y AKS son los tres grandes exponentes de este modelo, cada uno con su personalidad, ventajas y particularidades.

## Amazon EKS (Elastic Kubernetes Service)

EKS fue el ultimo de los tres grandes en llegar al mercado (2018), pero AWS compenso su tardanza con una integracion profunda en el ecosistema de servicios de Amazon. El plano de control de EKS se despliega en dos zonas de disponibilidad, con el API Server detras de un balanceador de carga gestionado por AWS, y etcd alojado en instancias dedicadas que Amazon opera y supervisa.

```bash
# Crear un cluster EKS con eksctl (herramienta oficial)
eksctl create cluster \
  --name mi-cluster \
  --region eu-west-1 \
  --nodegroup-name linux-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10 \
  --managed
```

EKS se integra nativamente con otros servicios de AWS:

- **ALB/NLB Ingress Controller**: balanceadores de carga de aplicacion y red gestionados por AWS.
- **IAM Roles for Service Accounts (IRSA)**: permisos finos para pods mediante roles IAM.
- **EBS CSI Driver**: volumenes persistentes respaldados por Elastic Block Store.
- **EFS CSI Driver**: almacenamiento compartido con Elastic File System.
- **VPC CNI**: red nativa de AWS, donde cada pod recibe una IP real de la VPC.
- **CloudWatch**: logs y metricas del cluster integrados con CloudWatch Logs y Container Insights.

> La red VPC CNI es una de las caracteristicas distintivas de EKS. A diferencia de otros proveedores donde los pods usan overlay networks, en EKS los pods tienen direcciones IP reales de la VPC, lo que simplifica la integracion con servicios como RDS o ElastiCache, que ven los pods como clientes con IPs directamente identificables en los logs.

```bash
# Asociar un rol IAM a un service account
eksctl create iamserviceaccount \
  --name mi-sa \
  --namespace default \
  --cluster mi-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

## Google GKE (Google Kubernetes Engine)

GKE es el servicio mas maduro de los tres. Google invento Borg, el predecesor de Kubernetes, y su experiencia en orquestacion de contenedores a escala planetaria se refleja en las caracteristicas avanzadas de GKE. El plano de control es gestionado por Google, con actualizaciones automaticas, replicacion entre zonas, y un SLA del 99.95% en configuraciones regionales.

```bash
# Crear un cluster GKE con gcloud
gcloud container clusters create mi-cluster \
  --region europe-west1 \
  --num-nodes 3 \
  --machine-type e2-standard-2 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10 \
  --enable-autorepair \
  --enable-autoupgrade
```

Las caracteristicas que distinguen a GKE:

- **Autopilot**: modo serverless donde Google gestiona los nodos por completo. El usuario solo define los pods y GKE decide donde y como ejecutarlos, eliminando la necesidad de gestionar node pools.
- **Horizontal Pod Autoscaler con metricas personalizadas**: GKE integra Stackdriver (Google Cloud Monitoring) como fuente de metricas para HPA, permitiendo escalado basado en cualquier metrica disponible en el proyecto.
- **Workload Identity**: equivalente a IRSA de AWS, permite asociar identidades de Google Cloud a service accounts de Kubernetes sin necesidad de gestionar credenciales manualmente.
- **GKE Sandbox**: ejecuta pods dentro de gVisor, un kernel de espacio de usuario que aísla los contenedores del kernel del host para entornos multi-tenant.
- **Config Connector**: gestiona recursos de Google Cloud (Cloud SQL, Pub/Sub, BigQuery) mediante manifiestos YAML de Kubernetes.

```bash
# Habilitar Autopilot en un cluster existente (migracion)
gcloud container clusters update mi-cluster --autopilot
```

> Autopilot representa un cambio de paradigma: el usuario ya no piensa en nodos, solo en pods. Google decide el tipo de instancia, el momento de escalado y la ubicacion optima. Esto simplifica la operacion pero elimina el control fino sobre la infraestructura subyacente, lo que puede ser problematico para cargas de trabajo con requisitos muy especificos de hardware.

## Azure AKS (Azure Kubernetes Service)

AKS llego al mercado entre EKS y GKE, y Microsoft ha priorizado la integracion con el ecosistema .NET y las herramientas de Microsoft, ademas de ofrecer un modelo de costos donde el plano de control es gratuito (solo se pagan los nodos workers).

```bash
# Crear un cluster AKS con az CLI
az aks create \
  --resource-group mi-grupo \
  --name mi-cluster \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10 \
  --enable-managed-identity
```

Diferenciadores de AKS:

- **Azure AD Integration**: los usuarios y service accounts de Kubernetes se autentican directamente contra Azure Active Directory, sin necesidad de gestionar certificados ni tokens de Kubernetes.
- **Virtual Node**: integracion con Azure Container Instances (ACI) que permite ejecutar pods sin aprovisionar nodos, util para picos de demanda repentinos.
- **Azure Policy**: aplica politicas de gobernanza (restricciones de seguridad, etiquetado, regiones permitidas) a nivel de cluster mediante Gatekeeper (OPA).
- **Kubernetes RBAC + Azure RBAC**: doble capa de control de acceso, combinando los roles nativos de Kubernetes con los roles de Azure.
- **Azure Monitor for Containers**: metricas, logs y trazas de los contenedores integradas con Azure Monitor.

```bash
# Habilitar Azure AD y RBAC en un cluster existente
az aks update \
  --resource-group mi-grupo \
  --name mi-cluster \
  --enable-aad \
  --enable-azure-rbac
```

## Comparativa

| Aspecto | EKS (AWS) | GKE (Google) | AKS (Azure) |
|---------|-----------|--------------|-------------|
| Costo plano de control | 0,10 USD/hora | 0,10 USD/hora (estandar) | Gratuito |
| SLA plano de control | 99,95% | 99,95% (regional) | 99,95% |
| Autopilot/Serverless | No nativo (Fargate) | Autopilot | Virtual Node (ACI) |
| Red de pods | VPC CNI (IP nativa) | VPC-native o alias IP | Azure CNI o Kubenet |
| Identidad | IRSA (IAM roles) | Workload Identity | Azure AD + Managed Identity |
| Actualizaciones | Manual o con eksctl | Automaticas (autoupgrade) | Automaticas (az aks upgrade) |
| Balanceador por defecto | Network Load Balancer | TCP/UDP Load Balancer | Azure Load Balancer |
| Almacenamiento | EBS, EFS, FSx | Persistent Disk, Filestore | Azure Disk, Azure Files |
| Windows containers | Soporte nativo | Experimental | Soporte nativo |
| Herramienta CLI | eksctl, aws CLI | gcloud, kubectl | az, kubectl |

## Criterios de seleccion

La eleccion entre EKS, GKE y AKS rara vez se basa en diferencias tecnicas de Kubernetes, ya que los tres implementan la misma API estandar. Las decisiones suelen estar determinadas por:

- **Ecosistema existente**: si la organizacion ya usa AWS, EKS ofrece la integracion mas natural con IAM, VPC, RDS y el resto de servicios. Lo mismo aplica para GCP con GKE y Azure con AKS.
- **Costo**: AKS tiene ventaja con el plano de control gratuito, pero los costos de los nodos workers, el almacenamiento y la transferencia de datos pueden variar significativamente entre proveedores.
- **Madurez de caracteristicas**: GKE suele recibir las ultimas versiones de Kubernetes primero y ofrece caracteristicas mas avanzadas (Autopilot, GKE Sandbox, Config Connector).
- **Cargas de trabajo especificas**: GKE tiene mejor soporte para GPUs y TPUs (ML/AI). EKS se integra mejor con herramientas de CI/CD de AWS (CodePipeline, CodeBuild). AKS ofrece la mejor integracion con el ecosistema .NET, Active Directory y SQL Server.
- **Soporte multicloud**: si la estrategia es multicloud, la portabilidad de Kubernetes mitiga parte del vendor locking, pero las integraciones nativas (IRSA, Workload Identity, Azure AD) son dificiles de replicar fuera del proveedor.

> El SLA del plano de control es solo una parte de la disponibilidad. La alta disponibilidad de las aplicaciones depende de como se configuren los nodos workers, las politicas de escalado, la distribucion entre zonas de disponibilidad y las estrategias de actualizacion. El plano de control gestionado elimina una fuente de fallos, pero no garantiza la disponibilidad de las cargas de trabajo.

## Consideraciones de migracion

Migrar entre proveedores cloud no es trivial, pero Kubernetes estandariza gran parte del camino. Los manifiestos YAML de deployments, services y configmaps son portables. Las diferencias sustanciales residen en:

- **StorageClass**: cada proveedor tiene sus propias clases de almacenamiento (gp2/gp3 en EKS, pd-standard/pd-ssd en GKE, managed-premium/managed-standard en AKS).
- **Ingress Controller**: los balanceadores de carga gestionados son diferentes (ALB en AWS, GLB en GCP, Azure Load Balancer en Azure).
- **Identidad y secretos**: IRSA, Workload Identity y Azure AD Managed Identity son mecanismos distintos que requieren cambios en la configuracion de los service accounts.
- **Red**: los rangos de IP de los pods, los CIDR de servicios y las politicas de red se configuran de forma diferente.

```yaml
# Ejemplo de StorageClass portable mediante parametrizacion
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ${STORAGE_PROVISIONER}
parameters:
  type: ${STORAGE_TYPE}
```

Donde `STORAGE_PROVISIONER` se resuelve a `ebs.csi.aws.com`, `pd.csi.storage.gke.io` o `disk.csi.azure.com` segun el entorno.

