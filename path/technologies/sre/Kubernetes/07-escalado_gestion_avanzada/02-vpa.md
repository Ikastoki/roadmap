# Vertical Pod Autoscaler

Mientras que el HPA ajusta el numero de replicas, el Vertical Pod Autoscaler (VPA) modifica los recursos asignados a cada Pod: CPU y memoria en los campos `requests` y `limits`. Es la herramienta adecuada cuando el problema no es el volumen de trafico sino la infra o sobredimension de los contenedores individuales. El VPA analiza el consumo historico y recomienda valores optimos, o los aplica directamente segun el modo de operacion configurado.

## El problema de los recursos estaticos

Al definir un Deployment, el desarrollador especifica `requests` (recursos garantizados) y `limits` (recursos maximos). Estos valores suelen ser estimaciones. Un request de CPU demasiado alto desperdicia capacidad del cluster; uno demasiado bajo causa throttling y degradacion del rendimiento. El VPA automatiza el ajuste fino que de otro modo requeriria ciclos repetitivos de monitorizacion, ajuste manual y redespliegue.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"
```

## Modos de operacion

El VPA soporta cuatro modos que determinan como actua sobre los Pods:

**Auto**: El VPA aplica las recomendaciones actualizando los Pods existentes. Para modificar los recursos de un Pod en ejecucion, Kubernetes debe recrearlo. El VPA utiliza el mecanismo de Eviction API: termina el Pod de forma controlada y el controlador del Deployment lo recrea con los nuevos valores de recursos. Esto puede causar una breve interrupcion si no hay replicas suficientes para absorber la recreacion.

**Recreate**: Similar a Auto, pero el VPA decide el momento exacto de la recreacion. El controlador del VPA solicita la expulsion del Pod y confia en que el controlador del recurso padre (Deployment, StatefulSet) lo reemplace con la configuracion actualizada. Es el modo recomendado para despliegues con multiples replicas, donde la interrupcion de un Pod no afecta al servicio global.

**Initial**: El VPA solo establece los recursos en el momento de creacion del Pod. Una vez en ejecucion, no modifica los valores aunque las recomendaciones cambien. Es util para aplicar sizing inicial sin intervencion manual, pero sin el riesgo de recreaciones durante la vida del Pod.

**Off**: El VPA unicamente genera recomendaciones sin aplicar cambios. Un operador humano puede revisarlas y decidir si aplicarlas manualmente. Es el punto de partida recomendado para adoptar VPA: observar las recomendaciones durante unos dias antes de activar Auto.

```bash
# Ver las recomendaciones del VPA en modo Off
kubectl get vpa api-vpa -o yaml
# En la seccion status.recommendation.containerRecommendations
```

## Recomendaciones y limitaciones

El VPA recomienda valores basandose en el percentil 90 del consumo historico. Esto significa que el Pod cubre el 90% de los escenarios de uso, sacrificando el 10% mas extremo en favor de la eficiencia. Para cargas de trabajo predecibles esta aproximacion funciona bien; para picos imprevisibles puede combinarse con HPA.

```yaml
status:
  recommendation:
    containerRecommendations:
    - containerName: api
      target:
        cpu: "250m"
        memory: "512Mi"
      lowerBound:
        cpu: "150m"
        memory: "384Mi"
      upperBound:
        cpu: "800m"
        memory: "1Gi"
      uncappedTarget:
        cpu: "500m"
        memory: "800Mi"
```

> El VPA no modifica los recursos de un Pod si el cambio esta dentro de un margen de tolerancia configurable. Esto evita recreaciones innecesarias por fluctuaciones minimas.

## VPA y HPA sobre CPU/memoria

Una restriccion importante: VPA y HPA no deben coexistir sobre las mismas metricas de recursos (CPU/memoria) en el mismo Deployment. Si el HPA escala basandose en CPU y el VPA modifica los requests de CPU, el HPA recibira lecturas inconsistentes y puede comportarse erraticamente. La solucion es usar HPA con metricas personalizadas y VPA para recursos base, o confiar en uno solo de los dos para el dimensionamiento vertical.

