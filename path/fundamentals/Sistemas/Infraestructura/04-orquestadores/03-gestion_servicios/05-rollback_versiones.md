# Rollback y control de versiones

Toda actualizacion en produccion conlleva un riesgo inherente: la nueva version puede introducir regresiones, errores de rendimiento o incompatibilidades que no se detectaron en pruebas. La capacidad de revertir rapidamente a una version anterior es el mecanismo de seguridad que permite a los equipos desplegar con confianza. Un orquestador moderno no solo almacena las revisiones historicas de cada recurso, sino que proporciona comandos nativos para ejecutar rollbacks de forma atomicas, restaurando el estado anterior sin necesidad de re-aplicar configuraciones manualmente ni intervenir en los nodos del clúster.

Kubernetes mantiene un historial de revisiones para cada Deployment. Cada vez que se modifica el manifest de un Deployment (ya sea directamente o mediante `kubectl apply`), el controlador crea una nueva revision numerada secuencialmente. Las revisiones se almacenan como ReplicaSets: cada revision corresponde a un ReplicaSet con una configuracion especifica de Pods. Cuando se ejecuta un rollback, el controlador ajusta la escala del ReplicaSet de la revision objetivo y reduce a cero el del ReplicaSet activo, replicando exactamente el estado que existia en ese punto del historial.

### Gestion basica de revisiones

```bash
# Ver las revisiones disponibles para un Deployment
kubectl rollout history deployment/api-gateway -n produccion
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl apply --filename=deployment.yaml --record=true
# 3         kubectl apply --filename=deployment.yaml --record=true

# Ver los detalles de una revision especifica
kubectl rollout history deployment/api-gateway --revision=2 -n produccion
```

> El flag `--record` en `kubectl apply` almacena el comando utilizado como causa del cambio en la anotacion `kubernetes.io/change-cause`. Sin esta anotacion, el historial de revisiones carece de contexto y es dificil determinar que version corresponde a cada revision.

### Ejecucion de rollback

```bash
# Revertir a la revision anterior automaticamente
kubectl rollout undo deployment/api-gateway -n produccion

# Revertir a una revision especifica
kubectl rollout undo deployment/api-gateway --to-revision=2 -n produccion

# Verificar el estado del rollback
kubectl rollout status deployment/api-gateway -n produccion

# Comprobar la revision actual
kubectl rollout history deployment/api-gateway -n produccion
```

### Rollback automatico por health check

Kubernetes puede abortar automaticamente una actualizacion si los nuevos Pods no pasan los health checks dentro de un tiempo razonable. Este comportamiento se configura con `progressDeadlineSeconds` en el bloque spec del Deployment. Si el controlador no detecta que el despliegue esta progresando (es decir, que los Pods nuevos pasan readiness) dentro de ese plazo, marca el rollout como `Progressing=False` con razon `ProgressDeadlineExceeded` y detiene la actualizacion. Sin embargo, esto no revierte automaticamente: deja de crear Pods nuevos, pero los Pods antiguos siguen en servicio. Para ejecutar el rollback automatico, se necesita un mecanismo externo (un controller, un pipeline de CI/CD o un script que observe el estado del Deployment y ejecute `kubectl rollout undo`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 4
  progressDeadlineSeconds: 300
  minReadySeconds: 30
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
          image: registry.empresa.com/api-gateway:v2.4.0
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
```

> `minReadySeconds` establece un periodo de gracia despues de que el Pod pasa el readiness: el controlador espera ese tiempo antes de considerar la replica como "lista". Esto evita que un Pod que falla instantaneamente despues de pasar el primer readiness check se conte como disponible, proporcionando una red de seguridad adicional contra despliegues de versiones inestables.

### Limitaciones del historial

El campo `revisionHistoryLimit` controla cuantas revisiones (ReplicaSets antiguos) se conservan. Por defecto, Kubernetes mantiene 10 revisiones. Si se establece un valor bajo (o 1), el historial se pierde rapido y la capacidad de rollback se reduce. En produccion, es recomendable mantener al menos 5 revisiones para tener margen de maniobra sin consumir recursos excesivos:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  revisionHistoryLimit: 5
```

> Un rollback ejecutado con `kubectl rollout undo` es, internamente, una nueva actualizacion del Deployment. Esto significa que se crea una nueva revision en el historial, no que se elimina la revision fallida. El historial crece con cada operacion de rollback, lo que es correcto: el historial debe reflejar todas las transiciones de estado, incluyendo las revertidas.

