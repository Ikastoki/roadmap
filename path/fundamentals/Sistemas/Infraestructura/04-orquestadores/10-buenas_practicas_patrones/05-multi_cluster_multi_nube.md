# Multi-cluster y multi-nube

Una sola instancia de Kubernetes, por bien configurada que este, presenta limitaciones ineludibles: un unico punto de fallo a nivel de region, restricciones de latencia para usuarios geograficamente dispersos, dependencia de un unico proveedor cloud, y limites de escalabilidad vertical del plano de control. La operacion multi-cluster y multi-nube aborda estos problemas distribuyendo la carga, los datos y el control across multiples clústeres, ya sea dentro de un mismo proveedor o atravesando multiples nubes.

Gestionar multiples clusters introduce complejidad significativa en areas como la red (como conectan los clusters entre si), la seguridad (como se distribuyen las identidades y credenciales), la observabilidad (como se unifican los logs y metricas), y la orquestacion (como se despliega y coordina el mismo workload across clusters). Las herramientas adecuadas transforman esta complejidad manejable en una ventaja operativa.

## Por que multi-cluster

Las razones para adoptar un modelo multi-cluster se agrupan en tres categorias fundamentales:

**Alta disponibilidad geografica**: un cluster en `eu-west-1` no protege contra un fallo de toda la region. Distribuir clusters en multiples zonas de disponibilidad o regiones garantiza que la caida de una no afecte a las demas.

**Cumplimiento y soberania de datos**: regulaciones como GDPR exigen que ciertos datos permanezcan dentro de la Union Europea. Multi-cluster permite ejecutar workloads especificos en regiones geograficas concretas sin comprometer la operacion global.

**Evitar vendor lock-in**: depender de un unico proveedor cloud crea riesgos strategicos y economicos. Multi-nube permite migrar workloads entre proveedores, negociar contratos con mayor flexibilidad, y aprovechar las fortalezas especificas de cada plataforma.

## Arquitectura multi-cluster

La arquitectura multi-cluster puede adoptar varias formas segun el patron de comunicacion entre clusters:

**Cluster Peering**: cada cluster se conecta directamente con los demas. Funciona para pocos clusters (hasta 5-7) pero escala mal combinatoriamente.

**Hub-and-Spoke**: un cluster central (hub) actua como punto de control y los demas (spokes) se conectan unicamente al hub. Reduce la complejidad de red pero concentra riesgo en el hub.

**Regional con global load balancing**: cada cluster opera de forma independiente con su propio plano de control, y un load balancer global (como AWS Global Accelerator o GCP Global Load Balancer) distribuye el trafico entre ellos.

```yaml
# Ejemplo: Cluster API para gestionar multiples clusters
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: cluster-eu-west
  namespace: capi-system
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.0.0.0/16"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: cluster-eu-west-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: cluster-eu-west
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: cluster-eu-west
  namespace: capi-system
spec:
  region: eu-west-1
  sshKeyName: default
  network:
    vpc:
      cidrBlock: "10.0.0.0/16"
    subnets:
      - availabilityZone: eu-west-1a
        cidrBlock: "10.0.1.0/24"
        isPublic: true
      - availabilityZone: eu-west-1b
        cidrBlock: "10.0.2.0/24"
        isPublic: true
```

## Rancher: gestion centralizada

Rancher es una plataforma de gestion multi-cluster que proporciona una interfaz unica para administrar multiples clústeres Kubernetes, independientemente de donde esten ejecutados: EKS, GKE, AKS, o clusters auto-gestionados. Rancher actua como un plano de control para los planes de control, ofreciendo vision unificada de recursos, seguridad y politicas across clusters.

```yaml
# Rancher Fleet: despliegue multi-cluster con GitOps
# fleet.yaml en el repositorio de la aplicacion
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: mi-aplicacion
  namespace: fleet-local
spec:
  repo: https://github.com/mi-org/gitops-repo
  branch: main
  paths:
    - clusters/eu-west
    - clusters/us-east
  targets:
    - name: eu-clusters
      clusterSelector:
        matchLabels:
          region: eu
    - name: us-clusters
      clusterSelector:
        matchLabels:
          region: us
  # Politicas de despliegue por grupo
  rolloutStrategy:
    name: default
    options:
      defaultTimeout: 600s
---
# Bundle: agrupa manifests para despliegue coordinado
apiVersion: fleet.cattle.io/v1alpha1
kind: BundleDeployment
metadata:
  name: mi-aplicacion-eu
  namespace: fleet-default
spec:
  stagedOptions:
    helm:
      release: mi-aplicacion
      values:
        replicaCount: 5
        region: eu-west
        image:
          tag: "1.2.0"
    kustomize:
      dir: overlays/eu
  targetRevision: main
```

```bash
# Registrar un cluster existente en Rancher
rancher cluster import mi-cluster-kubeconfig.yaml

# Ver todos los clusters gestionados
rancher clusters

# Desplegar una aplicacion en multiples clusters via CLI
rancher app install mi-chart --name mi-app \
  --target cluster-eu-west,cluster-us-east \
  --values values-multi.yaml

# Verificar estado de sincronizacion en todos los clusters
rancher fleets get --output json | jq '.[].conditions'
```

## Federacion de clusters

La federacion de clusters coordina la replicacion de recursos entre multiples clústeres. El modelo de federation garantiza que ciertos recursos (como namespaces, politicas de red, o configuraciones de seguridad) existan de forma consistente en todos los clusters participantes, mientras que otros recursos (como Deployments o Services) se despliegan selectivamente en clusters especificos segun criterios de negocio.

```yaml
# Karmada: sistema de federacion para Kubernetes
# Propagacion de politicas a multiples clusters
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: namespace-propagation
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Namespace
      name: produccion
  placement:
    clusterAffinity:
      clusterNames:
        - cluster-eu-west
        - cluster-us-east
        - cluster-ap-south
    replicaScheduling:
      type: Divided
      divide:
        byGroups:
          - groupName: eu
            clusterNames:
              - cluster-eu-west
            weight: 60
          - groupName: us
            clusterNames:
              - cluster-us-east
            weight: 40
---
# Override: personalizacion por cluster
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: image-override-per-region
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: backend
  overrideRules:
    - targetCluster:
        clusterNames:
          - cluster-eu-west
      overriders:
        imageOverrider:
          - operator: replace
            component: registry
            value: registry.eu.example.com
          - component: tag
            value: "1.2.0-eu"
    - targetCluster:
        clusterNames:
          - cluster-us-east
      overriders:
        imageOverrider:
          - operator: replace
            component: registry
            value: registry.us.example.com
          - component: tag
            value: "1.2.0-us"
```

```yaml
# Workload distribuido con Karmada
apiVersion: apps.karmada.io/v1alpha1
kind: MultiClusterDeployment
metadata:
  name: backend
  namespace: produccion
spec:
  template:
    metadata:
      labels:
        app: backend
    spec:
      replicas: 10
      selector:
        matchLabels:
          app: backend
      template:
        metadata:
          labels:
            app: backend
        spec:
          containers:
            - name: backend
              image: mi-org/backend:1.2.0
              ports:
                - containerPort: 8080
  placement:
    clusterAffinity:
      clusterSelector:
        matchLabels:
          tier: production
    replicaScheduling:
      type: Dynamic
      dynamic:
        weight: equal
        divisor:
          clusters:
            - cluster-eu-west
            - cluster-us-east
  conflicts:
    - resource:
        apiGroups: ["*"]
        kinds: ["*"]
      policy: Overwrite
```

## Observabilidad unificada

Un cluster aislado es relativamente sencillo de monitorear. Multiples clusters requieren una estrategia que consolide metricas, logs y trazas en un punto central sin perder la granularidad de los datos por cluster. La solucion tipica combina un recolector de datos distribuido (Fluent Bit o OpenTelemetry Collector en cada cluster) con un almacenamiento centralizado (Prometheus con Thanos o Cortex, Elasticsearch, Grafana Loki).

```yaml
# Thanos Sidecar: exportar metricas de Prometheus a un almacenamiento compartido
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:v2.48.0
          args:
            - --config.file=/etc/prometheus/prometheus.yml
            - --storage.tsdb.path=/prometheus
            - --storage.tsdb.retention.time=2h
        - name: thanos-sidecar
          image: quay.io/thanos/thanos:v0.33.0
          args:
            - sidecar
            - --tsdb.path=/prometheus
            - --objstore.config-file=/etc/thanos/bucket-config.yaml
          ports:
            - containerPort: 10901
              name: grpc
            - containerPort: 10902
              name: http
          volumeMounts:
            - name: thanos-config
              mountPath: /etc/thanos
      volumes:
        - name: thanos-config
          secret:
            secretName: thanos-objstore-config
```

```bash
# Verificar conectividad de multiples clusters con kubectx
kubectx cluster-eu-west
kubectl get nodes
kubectx cluster-us-east
kubectl get nodes
kubectx cluster-ap-south
kubectl get nodes

# Usar kubie para contextos aislados sin cambiar el kubeconfig global
kubie ctx cluster-eu-west -- kubectl get pods -A
kubie ctx cluster-us-east -- kubectl get pods -A
```

> La gestion multi-cluster no es un fin en si mismo. Adoptar esta arquitectura incrementa significativamente la complejidad operativa. Evalua cuidadosamente si los beneficios justifican el costo: para la mayoria de organizaciones, un cluster bien dimensionado con multiple zonas de disponibilidad es suficiente antes de escalar a multi-cluster.

