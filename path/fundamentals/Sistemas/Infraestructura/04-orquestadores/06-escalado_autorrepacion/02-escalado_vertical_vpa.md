# Escalado Vertical con VPA

Mientras el escalado horizontal varía el número de réplicas, el **Vertical Pod Autoscaler** (VPA) aborda un problema diferente pero igualmente critico: la asignacion inadecuada de recursos CPU y memoria a los contenedores. En la mayoria de los clusters, las especificaciones de recursos son estimaciones estáticas que se configuran una vez y rara vez se revisan. Esto genera un escenario comun donde los pods consumen una fracción de los recursos reservados (desperdicio de infraestructura) o bien exceden sus limites provocando throttling y OOMKills. El VPA analiza el patron de consumo real de cada pod y ajusta automaticamente las solicitudes y limites para alinearlos con la demanda efectiva.

El VPA opera a nivel de **Deployment**, **ReplicaSet** o **StatefulSet**, evaluando el historial de consumo de recursos de los pods gestionados por estos controladores. A diferencia del HPA, que toma decisiones cada pocos segundos, el VPA requiere un periodo de observacion mas extenso — tipicamente entre 8 y 24 horas — para construir un modelo estadistico confiable del patron de consumo. Esto se debe a que los recursos verticales no se pueden ajustar dinamicamente en un pod en ejecución: cada cambio de especificación requiere la recreación del pod, lo cual implica un reinicio del proceso de la aplicación.

## Modos de Operacion

El VPA ofrece tres modos de operacion que determinan el nivel de automatizacion en la gestion de recursos. El modo **Off** opera exclusivamente como sistema de recomendaciones sin aplicar cambios automaticos, siendo ideal para entornos donde se desea observar el comportamiento antes de automatizar. El modo **Initial** aplica las recomendaciones solo en la creacion de nuevos pods, preservando la configuracion existente durante el ciclo de vida del pod. El modo **Auto** aplica recomendaciones tanto a pods existentes (recreandolos) como a nuevos, representando la automatizacion completa del escalado vertical.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "api-server"
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2"
        memory: "4Gi"
      controlledResources: ["cpu", "memory"]
    - containerName: "sidecar-logger"
      mode: "Off"
  recommends:
  - targetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: backend-api
    lowerBound:
      cpu: "200m"
      memory: "256Mi"
    upperBound:
      cpu: "1"
      memory: "2Gi"
    target:
      cpu: "500m"
      memory: "1Gi"
```

> El modo `Auto` puede provocar interrupciones frecuentes en servicios criticos debido a la recreacion de pods. En entornos de produccion, se recomienda comenzar con `Off` para recopilar recomendaciones durante al menos una semana de trafico representativo antes de migrar a `Initial` o `Auto`.

## Diferencias con HPA

La decision entre VPA y HPA depende fundamentalmente de la naturaleza de la carga de trabajo. El HPA es apropiado para **workloads stateless** que pueden escalar horizontalmente añadiendo mas instancias, donde la carga se distribuye entre multiples copias. El VPA es ideal para **workloads con estado** o aplicaciones donde el rendimiento depende de la cantidad de recursos disponibles por instancia, como bases de datos, motores de busqueda o sistemas de procesamiento de grafos.

```bash
# verificacion de recomendaciones del VPA
kubectl get vpa -A
kubectl describe vpa backend-vpa -n production

# inspeccion de recomendaciones especificas
kubectl get vpa backend-vpa -o yaml

# monitoreo de cambios de recomendacion
kubectl get vpa backend-vpa --watch

# consulta de metricas de consumo real vs solicitado
kubectl top pods -n production --sort-by=cpu
```

> VPA y HPA **no pueden operar simultáneamente sobre la misma metrica** (CPU o memoria) del mismo Deployment. Si se necesita escalado vertical para recursos y horizontal para otra metrica, se puede configurar el HPA con metricas custom mientras el VPA gestiona CPU y memoria.

## Interaccion con QoS y Prioridad

El VPA interactúa directamente con el sistema de Quality of Service (QoS) de Kubernetes, que clasifica los pods en tres categorías: **Guaranteed**, **Burstable** y **BestEffort**. Cuando el VPA ajusta las solicitudes de recursos, puede alterar la clasificacion QoS del pod, afectando su prioridad en situaciones de presion de recursos del nodo. Un pod clasificado como Guaranteed tiene la mayor proteccion contra eviction, mientras que BestEffort es el primero en ser eliminado cuando el nodo queda sin memoria disponible.

```yaml
# ejemplo de como el VPA puede afectar la clasificacion QCS
# antes del VPA (Burstable):
resources:
  requests:
    cpu: "200m"      # menor que limits
    memory: "256Mi"  # menor que limits
  limits:
    cpu: "1"
    memory: "1Gi"

# despues del VPA (Guaranteed):
resources:
  requests:
    cpu: "500m"      # igual a limits
    memory: "512Mi"  # igual a limits
  limits:
    cpu: "500m"
    memory: "512Mi"
```

> Cuando el VPA establece `requests` iguales a `limits`, el pod se clasifica automaticamente como **Guaranteed**, proporcionando la maxima proteccion contra eviction pero eliminando la capacidad de burst. Esto puede ser problematico para workloads con patrones de consumo irregular.

## Implementacion y Configuracion del Recommender

El componente core del VPA es el **Recommender**, que utiliza algoritmos estadisticos para analizar el historial de consumo y predecir las necesidades futuras. El Recommender calcula recomendaciones basadas en percentiles — tipicamente el percentil 90 o 95 del consumo historico — para garantizar que la mayoria de las solicitudes de recursos sean satisfechas sin caer en throttling.

```bash
# instalacion de VPA a traves del repositorio vertical-pod-autoscaler
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# generacion de certificados TLS para el VPA
./hack/vpa-up.sh --component=vpa-recommender

# verificacion de los componentes del VPA
kubectl get pods -n kube-system -l app=vpa-recommender
kubectl get pods -n kube-system -l app=vpa-updater
kubectl get pods -n kube-system -l app=vpa-admission-controller

# logs del recommender para depuracion
kubectl logs -n kube-system -l app=vpa-recommender --tail=100
```

> El VPA-updater es responsable de evictar pods para que el controlador de replicas los recree con las nuevas especificaciones. En clusters con `PodDisruptionBudgets` configurados, el updater respetara las restricciones de disponibilidad, lo que puede retrasar la aplicacion de recomendaciones. Si el VPA necesita recrear un pod pero el PDB lo impide, la recomendacion quedara pendiente hasta que se cumplan las condiciones de disponibilidad.

