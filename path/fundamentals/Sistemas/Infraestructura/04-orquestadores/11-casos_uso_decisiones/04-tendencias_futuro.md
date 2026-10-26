# Tendencias y futuro de la orquestacion

La orquestacion de contenedores esta evolucionando mas alla de la gestion basica de ciclos de vida de pods y servicios. Las necesidades de las organizaciones estan empujando hacia capas de abstraccion superiores que resuelven problemas de complejidad operativa, seguridad de comunicaciones inter-servicio, ejecucion de carga de trabajo en el edge y la integracion nativa de modelos de inteligencia artificial dentro del pipeline de despliegue. Estas tendencias no reemplazan a Kubernetes, sino que construyen sobre el como plataforma, creando ecosistemas especializados para dominios concretos.

## Service Mesh

Un service mesh implementa una capa de infraestructura dedicada a gestionar toda la comunicacion entre servicios en un cluser orquestado. En lugar de que cada servicio implemente su propia logica de retries, circuit breakers, encriptacion y observabilidad de trafico, estas capacidades se externalizan a sidecars proxies (como Envoy) que se inyectan automaticamente en cada pod. El resultado es una red de servicios con trafico observable, seguro y resiliente sin modificar el codigo de la aplicacion.

Las dos implementaciones principales son Istio y Linkerd. Istio ofrece la funcionalidad mas amplia (traffic shaping avanzado, politicas de seguridad granulares, integracion con多种 backends de telemetria), pero su complejidad operativa y consumo de recursos es significativamente mayor. Linkerd prioriza la simplicidad y el rendimiento, con un sidecar mas ligero (linkerd2-proxy escrito en Rust) y una configuracion mas sencilla.

```yaml
# Istio: VirtualService con politica de trafico avanzada
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payments-service
  namespace: produccion
spec:
  hosts:
    - payments.miempresa.com
  http:
    - route:
        - destination:
            host: payments-v2
            subset: canary
          weight: 5
        - destination:
            host: payments-v2
            subset: estable
          weight: 95
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: 5xx,reset,connect-failure
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 5s
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payments-dr
  namespace: produccion
spec:
  host: payments-v2
  subsets:
    - name: estable
      labels:
        version: v2.1
    - name: canary
      labels:
        version: v2.2-rc1
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
```

```yaml
# Linkerd: configuracion mas declarativa y ligera
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: payments-service.produccion.svc.cluster.local
  namespace: produccion
spec:
  routes:
    - name: POST /api/v2/charge
      condition:
        method: POST
        pathRegex: /api/v2/charge
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
  retryBudget:
    retryRatio: 0.2
    minRetriesPerSecond: 10
    ttl: 10s
```

> La decision entre Istio y Linkerd no es solo tecnica. Istio requiere un equipo dedicado o experiencia significativa en operaciones. Linkerd puede ser operado por un equipo pequeno. El error comun es elegir Istio por su mayor funcionalidad sin considerar el coste operativo real de mantenerlo.

## Serverless en Kubernetes

Knative y OpenFaaS traen el modelo serverless a Kubernetes, permitiendo ejecutar funciones o servicios que escalan a cero cuando no reciben trafico. Esto resuelve uno de los problemas economicos de Kubernetes: mantener pods en ejecucion permanente consume recursos incluso cuando la aplicacion no esta sirviendo peticiones. Con Knative, un servicio que recibe una peticion cada hora solo consume recursos durante los milisegundos que tarda en procesar esa peticion.

Knative compone sobre Kubernetes tres componentes principales: Serving (escalamiento automatico incluyendo a cero), Eventing (enrutamiento de eventos entre servicios) y los reconcilers de build (construccion de imagenes nativa). La integracion con Kubernetes es total: Knative usa Serving, Services y Secrets nativos de Kubernetes bajo el capo.

```yaml
# Knative Service: escalamiento a cero automatico
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: procesador-pdf
  namespace: produccion
  annotations:
    autoscaling.knative.dev/minScale: "0"
    autoscaling.knative.dev/maxScale: "20"
    autoscaling.knative.dev/target: "10"
    autoscaling.knative.dev/scaleDownDelay: "30s"
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"
        autoscaling.knative.dev/metric: "concurrency"
    spec:
      containers:
        - image: mi-registry/procesador-pdf:v1.4.0
          ports:
            - containerPort: 8080
          env:
            - name: MAX_CONCURRENT_JOBS
              value: "5"
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
  traffic:
    - revisionName: procesador-pdf-v1
      percent: 100
```

```bash
# Desplegar el servicio Knative
kubectl apply -f procesador-pdf.yaml

# Verificar el escalado
kubectl get ksvc procesador-pdf -n produccion

# Ver las revisiones y trafico
kubectl get ksvc procesador-pdf -n produccion -o yaml
```

## Orquestacion en el edge

El edge computing requiere orquestar contenedores en dispositivos con recursos limitados, conexiones intermitentes y condiciones de red adversas. Los frameworks tradicionales de Kubernetes son demasiado pesados para un Raspberry Pi o un dispositivo industrial con 512MB de RAM. Soluciones como K3s, MicroK8s y KubeEdge adaptan Kubernetes a estos escenarios reduciendo la huella de memoria y habilitando la sincronizacion descentralizada.

KubeEdge extiende el plano de control de Kubernetes hacia los dispositivos edge, permitiendo que un cluser central gestione despliegues en miles de dispositivos remotos que operan con conectividad limitada. Los dispositivos pueden seguir funcionando autonomamente durante periodos de desconexion y sincronizan su estado cuando recuperan conectividad.

```bash
# K3s: Kubernetes ligero para edge (menos de 100MB de RAM)
# Instalacion en un dispositivo ARM64 (Raspberry Pi 4)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -

# Verificar el nodo
sudo k3s kubectl get nodes

# Desplegar una aplicacion en el edge
sudo k3s kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sensor-reader
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sensor-reader
  template:
    metadata:
      labels:
        app: sensor-reader
    spec:
      containers:
        - name: reader
          image: mi-registry/sensor-reader:arm64
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
            limits:
              cpu: "200m"
              memory: "64Mi"
EOF
```

## WebAssembly (Wasm) como alternativa a contenedores

WebAssembly esta emergiendo como una alternativa a los contenedores OCI en escenarios donde el tiempo de arranque, la huella de memoria y el aislamiento de seguridad son criticos. Un microservicio compilado a Wasm se inicia en microsegundos (frente a los cientos de milisegundos de un contenedor) y su binario pesa kilobytes en lugar de megabytes. El estandar WASI (WebAssembly System Interface) proporciona una capa de abstraccion del sistema operativo que permite ejecutar el mismo binario en multiples plataformas sin recompilacion.

WasmWorkloads es una API de Kubernetes que permite desplegar modulos Wasm como workloads nativos, gestionados por el mismo orquestador que ejecuta contenedores. RunWasm actua como un runtime que traduce las llamadas WASI del modulo Wasm a las llamadas del sistema operativo subyacente, habilitando el aislamiento de procesos sin la sobrecarga de un sistema de archivos virtual completo.

```yaml
# Wasm workload en Kubernetes con containerd + runWasm
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: wasmtime
handler: wasmtime
overhead:
  cpu: "50m"
  memory: "16Mi"
scheduling:
  nodeSelector:
    kubernetes.io/arch: amd64
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: filtro-imagenes
  namespace: produccion
spec:
  replicas: 2
  selector:
    matchLabels:
      app: filtro-imagenes
  template:
    metadata:
      labels:
        app: filtro-imagenes
    spec:
      runtimeClassName: wasmtime
      containers:
        - name: filtro
          image: mi-registry/filtro-imagenes:wasm32-wasi
          command: ["/filtro.wasm"]
          args: ["--port", "8080", "--mode", "resize"]
          resources:
            requests:
              cpu: "10m"
              memory: "8Mi"
            limits:
              cpu: "100m"
              memory: "32Mi"
```

> Wasm no reemplazara a los contenedores en el corto plazo para la mayoria de workloads. Su ecosistema de librerias sigue siendo limitado y no todas las dependencias del sistema operativo se modelan fielmente en WASI. Sin embargo, para funciones de procesamiento puro (filtros, transformaciones, validaciones), ofrece ventajas de rendimiento y seguridad claras.

## Orquestacion de carga de trabajo AI/ML

Los modelos de inteligencia artificial requieren recursos especificos (GPUs, TPUs, memoria HBM) y patrones de ejecucion distintos a los servicios web tradicionales: entrenamiento distribuido en multiples nodos, inferencia con modelos de multiples gigabytes y pipelines de datos que requieren acceso a almacenamiento de alto rendimiento. Kubeflow extiende Kubernetes con herramientas especificas para el ciclo de vida de los modelos de ML, desde el desarrollo en Jupyter notebooks hasta el despliegue de inferencia en produccion.

La gestion de GPUs en Kubernetes se realiza mediante device plugins que exponen los dispositivos de hardware como recursos schedulables. Los nodos con GPUs se etiquetan automaticamente y el scheduler asigna pods que requieran GPU solo a estos nodos. La comparticion de GPUs (NVIDIA MPS o time-slicing) permite ejecutar multiples modelos en una unica GPU cuando la carga de inferencia no la satura.

```yaml
# Kubeflow: Pipeline de entrenamiento distribuido con PyTorch
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: entrenamiento-deteccion
  namespace: ml-produccion
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
            - name: pytorch
              image: mi-registry/entrenamiento:v2.1
              command:
                - python
                - train.py
                - --epochs=50
                - --batch-size=32
                - --lr=0.001
              resources:
                limits:
                  nvidia.com/gpu: "4"
                  memory: "32Gi"
                  cpu: "8"
                requests:
                  memory: "16Gi"
                  cpu: "4"
              volumeMounts:
                - name: dataset
                  mountPath: /data
                  readOnly: true
                - name: modelos-output
                  mountPath: /output
          volumes:
            - name: dataset
              persistentVolumeClaim:
                claimName: dataset-deteccion
            - name: modelos-output
              persistentVolumeClaim:
                claimName: modelos-entrenados
    Worker:
      replicas: 3
      template:
        spec:
          containers:
            - name: pytorch
              image: mi-registry/entrenamiento:v2.1
              command:
                - python
                - train.py
              resources:
                limits:
                  nvidia.com/gpu: "4"
                  memory: "32Gi"
                  cpu: "8"
```

```yaml
# Autoscaler para inferencia con GPU (KEDA + NVIDIA)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: inferencia-escalado
  namespace: ml-produccion
spec:
  scaleTargetRef:
    name: modelo-inferencia
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: nvidia
      metadata:
        gpuCountPerInstance: "1"
      authenticationRef:
        name: nvidia-dcgm
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: inference_queue_length
        query: |
          sum(inference_queue_length{
            namespace="ml-produccion",
            service="modelo-inferencia"
          })
        threshold: "50"
```

