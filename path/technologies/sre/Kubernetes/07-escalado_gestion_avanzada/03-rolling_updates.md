# Rolling Updates y Rollbacks

Actualizar una aplicacion en produccion sin tiempo de inactividad es uno de los requisitos fundamentales de cualquier sistema orquestado. Kubernetes implementa las actualizaciones progresivas (rolling updates) como estrategia por defecto de los Deployments: los Pods antiguos se reemplazan gradualmente por nuevos, manteniendo el servicio disponible durante todo el proceso.

## Estrategia RollingUpdate

Cuando un Deployment se crea o modifica, su controlador compara el estado actual con el estado deseado y orquesta la transicion. Si la estrategia configurada es `RollingUpdate` (la opcion por defecto), el controlador reemplaza Pods de uno en uno o en lotes, segun los parametros definidos. La alternativa es `Recreate`, que elimina todos los Pods antiguos antes de crear los nuevos, causando tiempo de inactividad.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: api
        image: api:v2
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
```

## Parametros clave

**maxSurge**: define cuantos Pods adicionales pueden crearse por encima del numero deseado durante la actualizacion. Puede expresarse como entero absoluto o como porcentaje de las replicas. Con `maxSurge: 1` y 5 replicas, durante el update pueden coexistir hasta 6 Pods. Con `maxSurge: 25%`, se redondea hacia arriba: 2 Pods adicionales.

**maxUnavailable**: define cuantos Pods pueden estar no disponibles durante la actualizacion. Con `maxUnavailable: 0`, garantiza que siempre haya 5 Pods sirviendo trafico. Con `maxUnavailable: 1`, permite que un Pod se elimine antes de que el nuevo este listo, acelerando el proceso a costa de reducir temporalmente la capacidad.

La combinacion tipica para entornos productivos es `maxSurge: 1` y `maxUnavailable: 0`: minimo riesgo, velocidad moderada. Para entornos de desarrollo, `maxSurge: 25%` y `maxUnavailable: 25%` acelera las actualizaciones sin comprometer totalmente la disponibilidad.

```bash
# Iniciar una actualizacion
kubectl set image deployment/api api=api:v2

# Ver el progreso en tiempo real
kubectl rollout status deployment/api

# Pausar y reanudar
kubectl rollout pause deployment/api
kubectl rollout resume deployment/api
```

## Health checks y rollback automatico

El rolling update no es ciego. Cuando un nuevo Pod se crea, Kubernetes espera a que supere su readiness probe antes de considerar la actualizacion exitosa y proceder con el siguiente lote. Si el health check falla, Kubernetes detiene el rollout y marca la actualizacion como fallida.

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

El readiness probe es el guardian del rollout. Si el contenedor arranca pero no responde correctamente a las peticiones, Kubernetes no enruta trafico hacia el y no considera la actualizacion como completada. Tras tres fallos consecutivos, el rollout se detiene y los Pods antiguos permanecen ejecutandose.

```bash
# Ver el historial de cambios
kubectl rollout history deployment/api

# Volver a la version anterior
kubectl rollout undo deployment/api

# Volver a una version especifica
kubectl rollout undo deployment/api --to-revision=2
```

El mecanismo de revisiones permite deshacer cambios de forma inmediata. Cada cambio en la plantilla del Pod (imagen, variables de entorno, recursos) genera una nueva revision. El Deployment conserva las ultimas 10 revisiones por defecto, configurable mediante `spec.revisionHistoryLimit`.

> Un rollback no es instantaneo: sigue el mismo proceso gradual del rolling update, pero con las imagenes anteriores. Si se necesita una recuperacion rapida y hay suficientes replicas, puede ser preferible escalar el Deployment antiguo antes de hacer el rollback.

## Flujo completo de una actualizacion

Cuando se ejecuta `kubectl set image`, el Deployment crea un nuevo ReplicaSet con la nueva imagen. El controlador comienza a escalar hacia arriba el nuevo ReplicaSet y hacia abajo el antiguo, respetando maxSurge y maxUnavailable. Si en algun momento un Pod del nuevo ReplicaSet falla el readiness probe, el proceso se detiene. Solo cuando todos los Pods del nuevo ReplicaSet estan listos, el antiguo se reduce a cero.

```bash
# Ver los ReplicaSets involucrados
kubectl get replicasets -l app=api

# Observar el progreso paso a paso
kubectl describe deployment/api | tail -20
```

