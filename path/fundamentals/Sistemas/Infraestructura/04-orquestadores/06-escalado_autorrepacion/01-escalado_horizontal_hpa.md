# Escalado Horizontal con HPA

La capacidad de un servicio para absorber picos de demanda sin intervención manual constituye uno de los pilares fundamentales de la infraestructura moderna. El **Horizontal Pod Autoscaler** (HPA) de Kubernetes monitoriza continuamente las metricas asociadas a un conjunto de pods y ajusta dinámicamente el número de réplicas activas para mantener los objetivos de rendimiento definidos por el operador. A diferencia del escalado manual, donde un ingeniero decide cuántas instancias ejecutar, el HPA implementa un bucle de control cerrado que reacciona en segundos a variaciones de carga, escalando hacia arriba cuando los recursos se agotan y reduciendo réplicas cuando la demanda disminuye, optimizando así tanto la disponibilidad como el consumo de infraestructura.

El mecanismo interno del HPA se basa en la fórmula de cálculo de réplicas que compara el valor actual de la metrica objetivo con el valor deseado. Por ejemplo, si el objetivo de uso de CPU es 50% y el consumo actual promedio es de 80%, el controlador calcula `80 / 50 = 1.6`, redondeando hacia arriba para establecer el número de réplicas necesarias. Este cálculo se ejecuta cada 15 segundos por defecto (configurable mediante `--horizontal-pod-autoscaler-sync-period`), permitiendo una reacción ágil sin generar oscilaciones innecesarias.

## Metricas Soportadas

Kubernetes permite escalar basándose en diversas fuentes de datos. Las metricas nativas incluyen utilización de CPU y memoria, obtenidas directamente del servidor de metricas (`metrics-server`). Sin embargo, la verdadera potencia del HPA se despliega con las **metricas custom** y **externas**, que permiten escalar en función de parámetros específicos del negocio como solicitudes por segundo, longitud de cola de mensajes o latencia de respuesta.

```yaml
# metricas custom a traves de Prometheus Adapter
# se expone un endpoint /apis/custom.metrics.k8s.io/v1beta1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: gateway
        image: nginx:1.25
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

## Comportamiento y Periodos de Enfriamiento

Una de las problematicas mas criticas en cualquier sistema de escalado automatico es el fenomeno conocido como **flapping** o parpadeo, donde el sistema alterna continuamente entre escalar hacia arriba y hacia abajo debido a fluctuaciones transitorias de la carga. Para mitigar este problema, el HPA incorpora mecanismos de enfriamiento que imponen restricciones temporales sobre los cambios de escala.

El HPA de Kubernetes version 1.18+ introduce la sección `behavior` que permite definir con precision las politicas de escala. Los dos parametros clave son `scaleUp` y `scaleDown`, cada uno con sus propias restricciones de ventana temporal y estabilizacion.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
      - type: Pods
        value: 4
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 120
      selectPolicy: Min
```

> `selectPolicy: Max` en `scaleUp` permite el escalado mas agresivo entre las politicas definidas, mientras que `selectPolicy: Min` en `scaleDown` selecciona la politica mas conservadora, priorizando la estabilidad sobre la reduccion de costos.

La ventana de estabilizacion para escalar hacia abajo (`300 segundos`) es tipicamente mayor que la de escalado hacia arriba (`60 segundos`), reflejando la asimetria fundamental del escalado: es mejor escalar de mas antes que quedarse corto, pero reducir demasiado rapido puede provocar interrupciones de servicio.

## Requisitos e Instalacion

El HPA depende de una fuente de metricas para realizar sus calculos. En el caso de metricas nativas (CPU y memoria), se requiere el `metrics-server` instalado en el cluster. Para metricas custom basadas en Prometheus, se necesita `prometheus-adapter` o `keda` como puente entre Prometheus y la API de metricas de Kubernetes.

```bash
# verificacion de metricas disponibles
kubectl top nodes
kubectl top pods -n production

# instalacion del metrics-server (si no esta disponible)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# verificacion del HPA
kubectl get hpa -A
kubectl describe hpa api-gateway-hpa -n production

# monitoreo del escalado en tiempo real
kubectl get hpa api-gateway-hpa --watch
```

> El `metrics-server` debe estar correctamente configurado con los flags `--kubelet-preferred-address-types=InternalIP` y `--kubelet-insecure-tls` si el cluster no dispone de certificados TLS validos en los nodos worker. Esto es comun en entornos de desarrollo con herramientas como Minikube o Kind.

## Metricas Externas con KEDA

**KEDA** (Kubernetes Event-driven Autoscaling) extiende las capacidades del HPA permitiendo escalar basándose en eventos externos como la longitud de una cola de RabbitMQ, el lag de un topic de Kafka o el numero de mensajes pendientes en un service bus de Azure. KEDA actúa como un adaptador que consulta fuentes de datos externas y las convierte en metricas comprensibles para el HPA nativo de Kubernetes.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaledobject
  namespace: production
spec:
  scaleTargetRef:
    name: worker-deployment
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: rabbitmq
    metadata:
      host: "amqp://user:pass@rabbitmq.default.svc.cluster.local"
      queueName: "tasks"
      queueLength: "10"
  - type: prometheus
    metadata:
      serverAddress: "http://prometheus.monitoring.svc.cluster.local:9090"
      metricName: "http_requests_total"
      threshold: "100"
      query: |
        sum(rate(http_requests_total{service="api"}[1m]))
```

> KEDA instala dos componentes: un **Operator** que gestiona los objetos `ScaledObject` y un **Metrics Adapter** que expone las metricas al HPA. El pollingInterval determina la frecuencia con la que KEDA consulta las fuentes externas, impactando directamente la latencia de reaccion ante cambios de carga.

