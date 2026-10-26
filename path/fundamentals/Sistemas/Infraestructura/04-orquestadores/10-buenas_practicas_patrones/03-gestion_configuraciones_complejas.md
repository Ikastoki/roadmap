# Gestion de configuraciones complejas

A medida que las despliegues en Kubernetes crecen desde un par de manifests simples hasta cientos de recursos distribuidas en multiples entornos, la gestion manual de YAML se vuelve insostenible. Los valores de configuracion varian entre desarrollo, staging y produccion; las imagenes cambian con cada version; los recursos se multiplican por entorno. La gestion de configuraciones complejas aborda exactamente este problema: como mantener multiples variaciones de los mismos manifests de forma reproducible, versionable y auditada.

Existen tres enfoques principales, cada uno con fortalezas distintas: los empaquetadores con plantillas (Helm), los personalizadores declarativos (Kustomize), y las herramientas de infraestructura como codigo (Terraform, Pulumi) que gestionan tanto el cluster como sus recursos.

## Helm: el empaquetador de Kubernetes

Helm funciona como el gestor de paquetes de Kubernetes. Un chart de Helm es un directorio estructurado que contiene todos los manifests necesarios para desplegar una aplicacion, con plantillas que se parametrizan en tiempo de instalacion. El concepto es anologo a un paquete apt o yum, pero para recursos de Kubernetes.

Un chart contiene:

- **Chart.yaml**: metadatos del chart (nombre, version, dependencias).
- **values.yaml**: valores por defecto para las plantillas.
- **templates/**: manifests con sintaxis de Go templates.
- **charts/**: dependencias de otros charts.
- **helpers.tpl**: funciones auxiliares reutilizables.

```yaml
# Chart.yaml
apiVersion: v2
name: mi-aplicacion
description: Chart para la aplicacion backend
type: application
version: 0.1.0
appVersion: "1.2.0"
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```yaml
# values.yaml (valores por defecto)
replicaCount: 2

image:
  repository: mi-org/backend
  pullPolicy: IfNotPresent
  tag: "1.2.0"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: backend.local
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

postgresql:
  enabled: true
  auth:
    postgresPassword: ""

redis:
  enabled: true
  architecture: standalone
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mi-aplicacion.fullname" . }}
  labels:
    {{- include "mi-aplicacion.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mi-aplicacion.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mi-aplicacion.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: backend
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: DB_HOST
              value: "{{ .Release.Name }}-postgresql"
            - name: REDIS_HOST
              value: "{{ .Release.Name }}-redis"
```

```bash
# Instalar un chart con valores personalizados
helm install mi-app ./mi-aplicacion \
  --namespace production \
  --create-namespace \
  --values values-production.yaml

# Actualizar con nuevos valores
helm upgrade mi-app ./mi-aplicacion \
  --namespace production \
  --values values-production.yaml

# Ver historial de revisiones
helm history mi-app --namespace production

# Revertir a revision anterior
helm rollback mi-app 3 --namespace production
```

El historial de revisiones de Helm permite revertir cualquier despliegue a una version anterior con un unico comando. Esto es posible porque Helm almacena cada revision como un secreto en el namespace del release, manteniendo un registro completo de todos los manifests desplegados.

## Kustomize: personalizacion sin plantillas

Kustomize adopta un enfoque diferente: no usa plantillas ni logica de programacion. En su lugar, utiliza overlays (superposiciones) que modifican manifests base de forma declarativa. Es una herramienta nativa de `kubectl` (`kubectl apply -k`), lo que elimina la necesidad de instalar herramientas adicionales.

La estructura tipica de un proyecto Kustomize separa los manifests base de las variaciones por entorno:

```bash
mi-app/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
    configmap.yaml
  overlays/
    dev/
      kustomization.yaml
      replicas-patch.yaml
    staging/
      kustomization.yaml
      replicas-patch.yaml
    production/
      kustomization.yaml
      replicas-patch.yaml
      hpa.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: mi-app
  team: platform
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - hpa.yaml

namespace: production

namePrefix: prod-

commonLabels:
  environment: production

patches:
  - path: replicas-patch.yaml

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - CACHE_TTL=600
```

```yaml
# overlays/production/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 5
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```yaml
# overlays/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mi-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mi-app
  minReplicas: 5
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```bash
# Aplicar un overlay directamente
kubectl apply -k overlays/production/

# Previsualizar los manifests generados
kubectl kustomize overlays/production/

# Diferenciar con el cluster actual
kubectl diff -k overlays/production/
```

## Terraform y Pulumi: infraestructura como codigo

Mientras Helm y Kustomize gestionan los recursos dentro del cluster, Terraform y Pulumi gestionan el cluster en si mismo y su infraestructura subyacente: nodos, redes, load balancers, politicas IAM y los recursos de Kubernetes desde fuera del cluster. La distincion es fundamental: son herramientas de infraestructura como codigo (IaC) que orquestan la creacion y gestion de clusters y sus dependencias.

Terraform utiliza HCL (HashiCorp Configuration Language) y trabaja con providers para interactuar con multiples plataformas. El provider `kubernetes` permite declarar recursos de K8s, pero tambien existen providers para AWS EKS, GCP GKE, Azure AKS, y muchos mas.

```hcl
# main.tf - Configuracion de un cluster EKS con recursos Kubernetes
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }

  backend "s3" {
    bucket = "terraform-state-eks"
    key    = "cluster/terraform.tfstate"
    region = "eu-west-1"
  }
}

provider "aws" {
  region = var.aws_region
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "19.21.0"
  cluster_name    = var.cluster_name
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    general = {
      min_size     = 2
      max_size     = 10
      desired_size = 3

      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"
    }

    spot = {
      min_size     = 0
      max_size     = 5
      desired_size = 2

      instance_types = ["t3.large", "t3a.large"]
      capacity_type  = "SPOT"
    }
  }
}

resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
    labels = {
      environment = "production"
      managed_by  = "terraform"
    }
  }
}

resource "kubernetes_deployment" "backend" {
  metadata {
    name      = "backend"
    namespace = kubernetes_namespace.production.metadata[0].name
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "backend"
      }
    }

    template {
      metadata {
        labels = {
          app = "backend"
        }
      }

      spec {
        container {
          name  = "backend"
          image = "mi-org/backend:1.2.0"

          port {
            container_port = 8080
          }

          resources {
            limits = {
              cpu    = "500m"
              memory = "256Mi"
            }
            requests = {
              cpu    = "100m"
              memory = "128Mi"
            }
          }
        }
      }
    }
  }
}
```

```bash
# Inicializar Terraform (descargar providers)
terraform init

# Planificar cambios
terraform plan -var="cluster_name=mi-cluster"

# Aplicar cambios
terraform apply -var="cluster_name=mi-cluster"

# Ver estado actual
terraform state list

# Destruir toda la infraestructura
terraform destroy -var="cluster_name=mi-cluster"
```

Pulumi adopta un enfoque similar pero utiliza lenguajes de programacion generales (TypeScript, Python, Go, C#) en lugar de un DSL propio. Esto permite reutilizar logica de programacion existente, escribir tests unitarios, y aprovechar el ecosistema de paquetes de cada lenguaje.

> Terraform y Pulumi no sustituyen a Helm o Kustomize, sino que complementan su gestion. Un flujo tipico usa Terraform para crear el cluster EKS/GKE y sus dependencias de red, y luego Helm o Kustomize para desplegar las aplicaciones dentro del cluster.

