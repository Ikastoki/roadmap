# Entornos gestionados en nube

Los clusteres locales de desarrollo satisfacen la necesidad de iterar rapidamente durante el ciclo de codificacion, pero llega un momento en que la aplicacion necesita ejecutarse en una infraestructura que reproduzca las condiciones reales de produccion: alta disponibilidad, escalado automatico, integracion con servicios de almacenamiento, redes privadas, identidad gestionada y politicas de seguridad corporativas. Los proveedores de nube gestionan la complejidad operativa de mantener un cluster de Kubernetes funcionando — actualizaciones del plano de control, parches de seguridad, backup de etcd, balanceo del API server — permitiendo a los equipos centrarse exclusivamente en las cargas de trabajo. Esta abstraccion tiene un coste economico y una perdida de control granular, pero para la mayoria de organizaciones el balance es claramente favorable.

## Managed Kubernetes: proveedores principales

### Amazon EKS (Elastic Kubernetes Service)

EKS es el servicio gestionado de Kubernetes de AWS. Ejecuta un plano de control multi-AZ ( Availability Zone) gestionado por AWS, con un costo fijo por hora por cluster mas los costes de los nodos worker que se ejecutan en EC2 o Fargate (serverless). La integracion nativa con servicios AWS — IAM para RBAC, ALB/NLB para ingress, EBS/EFS para almacenamiento, CloudWatch para observabilidad — lo convierte en la opcion natural para organizaciones ya establecidas en el ecosistema AWS.

#### Creacion de cluster EKS con eksctl

```bash
# Instalar eksctl (CLI oficial de AWS para EKS)
# macOS
brew install eksctl

# Linux
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Crear cluster con configuracion basica
eksctl create cluster \
  --name mi-cluster \
  --region eu-west-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

# Verificar cluster
eksctl get cluster --region eu-west-1

# Obtener kubeconfig
eksctl utils write-kubeconfig --cluster mi-cluster --region eu-west-1
```

#### Configuracion avanzada con fichero

```yaml
# cluster-eks.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: produccion
  region: eu-west-1
  version: "1.29"

availabilityZones:
  - eu-west-1a
  - eu-west-1b
  - eu-west-1c

iam:
  withOIDC: true

managedNodeGroups:
  - name: general
    instanceType: m5.large
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 100
    volumeType: gp3
    labels:
      workload: general
    tags:
      Environment: production
    iam:
      attachPolicyARNs:
        - arn:aws:iam::policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::policy/AmazonEC2ContainerRegistryReadOnly

  - name: compute-optimized
    instanceType: c5.xlarge
    minSize: 0
    maxSize: 20
    desiredCapacity: 0
    labels:
      workload: compute
    taints:
      - key: workload
        value: compute
        effect: NoSchedule

cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler
```

```bash
# Crear cluster desde fichero
eksctl create cluster -f cluster-eks.yaml

# Escalar manualmente un nodegroup
eksctl scale nodegroup --cluster produccion --name general --nodes 5

# Actualizar version de Kubernetes
eksctl upgrade cluster --name produccion --region eu-west-1

# Eliminar cluster (ELIMINA TODO: nodos, volumes,负载)
eksctl delete cluster --name produccion --region eu-west-1
```

> El costo base de EKS es de aproximadamente 0.10 USD/hora por cluster (~73 USD/mes) mas los costes de los nodos EC2, elastic load balancers, almacenamiento y transferencia de datos. Para clusters pequeños de desarrollo, considerar Fargate como capa de nodos para eliminar la gestion de instancias EC2.

### Google GKE (Google Kubernetes Engine)

GKE es el servicio gestionado de Kubernetes de Google Cloud y se considera el mas maduro del mercado, dado que Google fue el creador original de Kubernetes. Ofrece variantes como GKE Autopilot (donde Google gestiona completamente los nodos) y GKE Standard (donde el usuario tiene control sobre los nodos). La integracion con Google Cloud Build, Artifact Registry, Cloud Monitoring y Cloud Logging proporciona un flujo de trabajo completo.

```bash
# gcloud CLI (requiere Google Cloud SDK instalado)
# Crear cluster GKE
gcloud container clusters create mi-cluster \
  --zone europe-west1-b \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --enable-autoscaling --min-nodes 1 --max-nodes 10 \
  --enable-autorepair --enable-autoupgrade \
  --release-channel regular

# Crear cluster Autopilot (sin gestion de nodos)
gcloud container clusters create-auto mi-cluster \
  --region europe-west1

# Obtener credenciales
gcloud container clusters get-credentials mi-cluster --zone europe-west1-b

# Verificar
kubectl get nodes
```

### Azure AKS (Azure Kubernetes Service)

AKS gestiona el plano de control de Kubernetes en Azure con integracion nativa con Azure Active Directory, Azure Monitor, Azure Policy y Azure Container Registry. Una caracteristica distintiva es que el plano de control es gratuito (solo se cobran los nodos worker), lo que lo economico para clusters pequeños.

```bash
# Azure CLI
# Crear grupo de recursos
az group create --name mi-rg --location westeurope

# Crear cluster AKS
az aks create \
  --resource-group mi-rg \
  --name mi-cluster \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --enable-addons monitoring

# Obtener credenciales
az aks get-credentials --resource-group mi-rg --name mi-cluster

# Verificar
kubectl get nodes

# Escalar nodos
az aks scale --resource-group mi-rg --name mi-cluster --node-count 5
```

### DigitalOcean DOKS

DOKS ofrece una experiencia simplificada de Kubernetes con precios predecibles y sin costes ocultos por el plano de control. Es popular entre startups y proyectos de tamano pequeno-medio que buscan la potencia de Kubernetes sin la complejidad de los grandes proveedores.

```bash
# doctl CLI
# Crear cluster DOKS
doctl kubernetes cluster create mi-cluster \
  --region fra1 \
  --size s-4vcpu-8gb \
  --count 3 \
  --version 1.29.1-do.0

# Obtener kubeconfig
doctl kubernetes cluster kubeconfig show mi-cluster > ~/.kube/doks-config.yaml
kubectl --kubeconfig=~/.kube/doks-config.yaml get nodes
```

## Managed Kubernetes: tabla comparativa

| Caracteristica | EKS (AWS) | GKE (Google) | AKS (Azure) | DOKS (DO) |
|---|---|---|---|---|
| Coste control plane | ~73 USD/mes | ~73 USD/mes | Gratis | Gratis |
| Autopilot/Serverless | Fargate | Autopilot | Virtual Nodes | No |
| Versiones K8s | 5 versiones | 4 versiones | 4 versiones | 3 versiones |
| Actualizaciones auto | Si (control plane) | Si (control plane + nodos) | Si (control plane) | Si |
| RBAC + OIDC | IAM + OIDC provider | IAM + Workload Identity | Azure AD | RBAC basico |
| Ingress gestionado | ALB/NLB | GCLB | Azure LB | LB gratuito |
| Almacenamiento | EBS, EFS | Persistent Disk, Filestore | Managed Disks, Azure Files | Volumes |
| Certificacion CNCF | Si | Si | Si | Si |

## Alternativas: Nomad y ECS

No todas las cargas de trabajo requieren la potencia (y la complejidad) de Kubernetes. HashiCorp Nomad y Amazon ECS proporcionan orquestacion de contenedores con una curva de aprendizaje significativamente menor.

### Nomad como alternativa

```hcl
# Nomad puede orquestar Docker, Podman, Java, QEMU, exec, y mas
# con una unica interfaz y un agente unico por nodo
job "api" {
  datacenters = ["dc1"]
  type = "service"

  group "backend" {
    count = 3

    network {
      port "http" { to = 8080 }
    }

    service {
      name = "api"
      port = "http"

      check {
        type     = "http"
        path     = "/health"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "server" {
      driver = "docker"
      config {
        image = "mi-app:latest"
        ports = ["http"]
      }
      resources {
        cpu    = 500
        memory = 512
      }
    }
  }
}
```

Nomad se despliega como un cluster de uno o varios nodos con un agente en cada uno. El plano de control es minimo (un solo binario) y el protocolo de consensus (Raft) se ejecuta entre los serviders. Para entornos de desarrollo, Nomad Agent Dev proporciona un cluster funcional en un solo comando sin dependencias externas.

### Amazon ECS

```bash
# Crear cluster ECS con Fargate (serverless)
aws ecs create-cluster --cluster-name mi-cluster

# Registrar la imagen en ECR
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.eu-west-1.amazonaws.com
docker tag mi-app:latest <ACCOUNT_ID>.dkr.ecr.eu-west-1.amazonaws.com/mi-app:latest
docker push <ACCOUNT_ID>.dkr.ecr.eu-west-1.amazonaws.com/mi-app:latest
```

> ECS es ideal si tu stack ya esta en AWS y no necesitas la portabilidad que ofrece Kubernetes. La falta de un estandar externo (como la API de Kubernetes) es su principal desventaja: migrar de ECS a otro sistema requiere reescribir las definiciones de tarea, mientras que con Kubernetes las manifests son portables entre proveedores.

### Comparativa Nomad vs ECS vs Kubernetes

| Aspecto | Nomad | ECS | Kubernetes |
|---|---|---|---|
| Complejidad operativa | Baja | Baja-Media | Alta |
| Multi-cloud | Si | No (AWS only) | Si (portable) |
| Workloads soportados | Multi-runtime | Docker/Fargate | Contenedores |
| Ecosistema de plugins | Moderado | Limitado | Extensivo |
| Curva de aprendizaje | Suave | Suave | Pronunciada |
| Coste de gestion | Minimo | Bajo (Fargate) | Alto |

