# Limites de Recursos

Un clúster de Kubernetes comparte recursos computacionales entre múltiples workloads que compiten por la misma capacidad de CPU, memoria y almacenamiento. Sin mecanismos de restricción, un proceso mal configurado o un componente en bucle infinito puede consumir la totalidad de los recursos de un nodo, afectando a todas las demás unidades de trabajo que se ejecutan en él. Los limites de recursos establecen las reglas del juego: cuántos recursos garantiza el orquestador para cada pod (requests) y cuánto puede consumir como máximo (limits).

## Requests: recursos garantizados

Los requests definen la cantidad mínima de recursos que el orquestador reserva para un contenedor. Cuando se crea un pod, el programador de Kubernetes utiliza los requests para decidir en qué nodo colocar el contenedor, verificando que el nodo tenga capacidad suficiente. Un contenedor nunca recibirá menos recursos que su request, incluso si el sistema está sobrecargado.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-backend
  namespace: produccion
spec:
  containers:
    - name: app
      image: empresa/api-backend:v2.1
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
          ephemeral-storage: "1Gi"
        limits:
          cpu: "1000m"
          memory: "512Mi"
          ephemeral-storage: "2Gi"
```

> Los valores de CPU se expresan en milicore: `500m` equivale a medio núcleo, `1000m` a un núcleo completo y `2000m` a dos núcleos. La memoria se expresa en bytes con sufijos: `Mi` (mebibytes), `Gi` (gibibytes), `Ki` (kibibytes).

## Limits: consumo maximo

Los limits establecen el techo de recursos que un contenedor puede utilizar. Si un contenedor excede su limit de memoria, el kernel del sistema operativo lo terminate con un código OOMKilled (Out of Memory Killed). Si excede su limit de CPU, el proceso no se termina pero se le restringe el tiempo de ejecución mediante el controlador CFS (Completely Fair Scheduler) de Linux.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-procesamiento
  namespace: produccion
spec:
  replicas: 3
  selector:
    matchLabels:
      app: worker-procesamiento
  template:
    metadata:
      labels:
        app: worker-procesamiento
    spec:
      containers:
        - name: worker
          image: empresa/worker:v1.5
          resources:
            requests:
              cpu: "1000m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "1Gi"
          env:
            - name: JAVA_OPTS
              value: "-Xmx768m -Xms256m"
```

> La configuración de `-Xmx` (memoria máxima del heap de Java) debe ser significativamente menor que el limit de memoria del contenedor. Si el heap de Java más la memoria off-heap exceden el limit de memoria del contenedor, el proceso será terminado con OOMKilled. Un margen de al menos 256Mi entre `-Xmx` y el memory limit es recomendado para cubrir metadatos del JVM y memoria nativa.

## Calidad de Servicio (QoS)

Kubernetes clasifica cada pod en una de tres clases de Calidad de Servicio según la configuración de requests y limits. Esta clasificación determina la prioridad de evacuación cuando un nodo se queda sin recursos: los pods con menor prioridad son los primeros en ser terminados.

### Clase Guaranteed

Un pod pertenece a la clase `Guaranteed` cuando cada contenedor tiene requests iguales a sus limits para CPU y memoria. Estos pods reciben la mayor prioridad y son los últimos en ser terminados durante un evento de presión de recursos. Representan workloads críticos de producción que requieren estabilidad absoluta.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: base-datos-critica
  namespace: produccion
  labels:
    app: postgres
spec:
  containers:
    - name: postgres
      image: postgres:16-alpine
      resources:
        requests:
          cpu: "2000m"
          memory: "4Gi"
        limits:
          cpu: "2000m"
          memory: "4Gi"
      env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
```

### Clase Burstable

Un pod es `Burstable` cuando tiene requests y/o limits definidos pero no son iguales en todos los contenedores para todos los recursos. Estos pods pueden consumir recursos adicionales cuando están disponibles, pero serán terminados antes que los pods `Guaranteed` en situaciones de presión.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-rest
  namespace: produccion
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api-rest
  template:
    metadata:
      labels:
        app: api-rest
    spec:
      containers:
        - name: app
          image: empresa/api:v3.0
          resources:
            requests:
              cpu: "250m"
              memory: "128Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
```

> La clase `Burstable` es la más común en producción. Permite que las aplicaciones se beneficien de recursos ociosos sin comprometer la estabilidad de los pods `Guaranteed`. El uso de requests bajos y limits más altos es un patrón habitual para servicios web con tráfico variable.

### Clase BestEffort

Un pod es `BestEffort` cuando ningún contenedor tiene requests ni limits definidos. Estos pods reciben la menor prioridad y son los primeros en ser terminados. No tienen garantía de recursos y se ejecutan únicamente cuando hay capacidad ociosa disponible.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: herramienta-debug
  namespace: desarrollo
  labels:
    app: debug-tools
spec:
  containers:
    - name: debug
      image: alpine:3.19
      command: ["sleep", "infinity"]
```

> Los pods `BestEffort` son útiles para herramientas de desarrollo, debugging o procesos batch de baja prioridad. Sin embargo, deben evitarse en producción porque no tienen garantía de estabilidad y pueden ser terminados en cualquier momento bajo presión de recursos.

## Limites por namespace

Para evitar que un namespace consuma una proporción desmedida de los recursos del clúster, se pueden definir ResourceQuotas que establezcan límites aggregados y LimitRanges que impongan defaults y rangos permitidos para los contenedores.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-produccion
  namespace: produccion
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "30"
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limites-default
  namespace: produccion
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "250m"
        memory: "128Mi"
      min:
        cpu: "100m"
        memory: "64Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      maxLimitRequestRatio:
        cpu: "4"
        memory: "2"
```

> El `LimitRange` actúa como una plantilla por defecto: si un contenedor no define sus propios requests y limits, el `LimitRange` les asigna los valores configurados en `default` y `defaultRequest`. Esto garantiza que ningún contenedor se ejecute sin restricciones de recursos, incluso si el desarrollador olvida especificarlas.

```bash
# Verificar la clase QoS de un pod
kubectl get pod <nombre-pod> -o jsonpath='{.metadata.labels}'

# Ver el uso real de recursos de un pod
kubectl top pod <nombre-pod> --namespace=produccion

# Ver el consumo de recursos del namespace
kubectl describe resourcequota quota-produccion -n produccion

# Ver el LimitRange aplicado
kubectl describe limitrange limites-default -n produccion
```

> `kubectl top` requiere que el Metrics Server esté instalado en el clúster. Sin él, los comandos de métricas no devuelven datos. El Metrics Server es un componente ligero que recopila métricas de uso de recursos de los nodos y pods y las expone a través de la API de Kubernetes.

