# Horizontal Pod Autoscaler

Una aplicacion desplegada con un numero fijo de replicas funciona correctamente bajo carga constante, pero en entornos reales el trafico fluctua. El Horizontal Pod Autoscaler (HPA) resuelve este problema ajustando automaticamente el numero de replicas de un Deployment, StatefulSet o ReplicaSet en funcion de metricas observables. Es la pieza que permite que un servicio consuma exactamente los recursos que necesita en cada momento, sin intervencion manual.

## Funcionamiento interno

El HPA se implementa como un controlador dentro del kube-controller-manager que evalua ciclicamente (por defecto cada 15 segundos) las metricas configuradas y calcula el numero deseado de replicas. El algoritmo base es:

```
desiredReplicas = ceil(currentReplicas * (currentMetricValue / desiredMetricValue))
```

Si el valor actual de CPU es del 80% y el objetivo es del 50%, con 4 replicas actuales, el calculo produce `ceil(4 * (80 / 50)) = ceil(6.4) = 7` replicas. El HPA no actua instantaneamente: incorpora mecanismos de estabilizacion para evitar oscilaciones (flapping). Por defecto, espera un periodo de cool-down antes de reducir replicas (5 minutos) y un periodo de estabilizacion antes de escalar.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

> El HPA v1 (autoscaling/v1) solo soporta CPU. La version v2 (autoscaling/v2) anadio memoria y metricas personalizadas. Siempre que sea posible, utiliza `autoscaling/v2`.

## Metricas soportadas

El HPA puede consumir metricas de tres origenes:

- **Resource metrics**: CPU y memoria por Pod, proporcionadas por metrics-server. Son las mas comunes y no requieren configuracion adicional.
- **Custom metrics**: metricas arbitrarias expuestas por la aplicacion a traves de un adaptador como Prometheus Adapter o Google Stackdriver. Por ejemplo, peticiones por segundo, longitud de cola o latencia.
- **External metrics**: metricas provenientes de servicios externos al cluster, como el numero de mensajes en una cola SQS o el tamano de un topic de Kafka.

```yaml
# Ejemplo con metrica personalizada
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: 100
```

## Requisito: metrics-server

El HPA no funciona sin un origen de metricas. El componente minimo necesario es metrics-server, un agregador de metricas de recursos que recolecta datos de CPU y memoria de cada nodo a traves de la API de kubelet.

```bash
# Instalacion de metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verificar que recoge metricas
kubectl top nodes
kubectl top pods
```

Metrics-server no almacena historico: solo mantiene los valores actuales en memoria. Para metricas personalizadas o historicas es necesario utilizar Prometheus con prometheus-adapter.

## kubectl autoscale

El comando `kubectl autoscale` ofrece una forma rapida de crear un HPA desde la CLI sin escribir un manifiesto YAML:

```bash
kubectl autoscale deployment api \
  --min=2 \
  --max=10 \
  --cpu-percent=50
```

Este comando genera un HPA equivalente al manifiesto anterior. Es util para entornos de desarrollo o para prototipado rapido, pero en produccion se prefiere el manifiesto declarativo versionado en el repositorio.

## Comportamiento avanzado

El HPA permite configurar politicas de escalado diferenciadas para subida y bajada mediante el campo `behavior`:

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

Esta configuracion elimina la ventana de estabilizacion para escalar hacia arriba (respuesta inmediata a picos de carga) y limita la reduccion a un Pod cada 60 segundos, evitando bajadas bruscas que podrian degradar la experiencia del usuario.

