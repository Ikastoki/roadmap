# Deployments

Un Deployment es el controlador mas utilizado en Kubernetes para gestionar aplicaciones stateless. Su funcion es declarar el estado deseado de un conjunto de pods y mantenerlo en el tiempo: si un pod falla, lo recrea; si se actualiza la imagen, realiza un rollout controlado; si se incrementa el numero de replicas, las crea. El Deployment abstrae completamente la gestion individual de los pods y proporciona los mecanismos necesarios para operar aplicaciones en produccion.

Detras de cada Deployment opera un ReplicaSet, que es el responsable directo de mantener el numero de pods en ejecucion. El Deployment gestiona los ReplicaSets como capa de versionado: cada vez que se modifica la especificacion del pod template, el Deployment crea un nuevo ReplicaSet y escala el anterior a cero, permitiendo realizar rollbacks a cualquier version anterior.

## Estructura de un manifiesto Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: mi-api:2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

El manifiesto contiene los siguientes campos clave:

**apiVersion**: `apps/v1` es la version estable de la API de Deployments. Las versiones anteriores (`extensions/v1beta1`, `apps/v1beta1`) estan obsoletas y no deben usarse.

**replicas**: el numero deseado de copias del pod. Kubernetes mantiene exactamente este numero de pods en ejecucion. Si no se especifica, el valor por defecto es 1.

**selector**: define como el Deployment encuentra los pods que gestiona. Debe coincidir con las labels definidas en el template. El selector no se puede modificar una vez creado el Deployment.

**template**: la especificacion del pod que se replicara. Es identica a la especificacion de un recurso Pod, pero anidada dentro del Deployment. Contiene su propio `metadata` con labels (que deben coincidir con el selector) y el `spec` del contenedor.

## Gestion de replicas

El ReplicaSet subyacente se encarga de crear y destruir pods para alcanzar el numero deseado de replicas. Cuando se crea un Deployment, Kubernetes crea automaticamente el ReplicaSet y este a su vez crea los pods.

```bash
# Crear un deployment
kubectl apply -f deployment.yaml

# Ver los recursos creados
kubectl get deployments
kubectl get replicasets
kubectl get pods

# Escalar a mas replicas
kubectl scale deployment api-deployment --replicas=5

# Escalar de forma declarativa (modificando el YAML y reaplicando)
kubectl apply -f deployment.yaml  # con replicas: 5

# Ver el estado del escalado
kubectl rollout status deployment api-deployment
```

Si un pod muere (el contenedor falla, el nodo se cae, o alguien lo borra accidentalmente), el ReplicaSet detecta que el numero actual de pods es menor que el deseado y crea un reemplazo inmediatamente.

```bash
# Simular la muerte de un pod
kubectl delete pod api-deployment-6f8d4c8b7c-abc12

# El ReplicaSet crea uno nuevo automaticamente
kubectl get pods --watch
```

## Estrategias de actualizacion

El campo `strategy` define como se actualizan los pods cuando se modifica el template del Deployment. Kubernetes soporta dos estrategias:

**RollingUpdate** (por defecto): reemplaza los pods gradualmente, manteniendo la aplicacion disponible durante todo el proceso. Se controla mediante dos parametros:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

`maxSurge` define el numero maximo de pods adicionales que pueden crearse durante la actualizacion. Puede ser un numero absoluto o un porcentaje (redondeado hacia arriba). Con `maxSurge: 1`, durante el rollout habra como maximo `replicas + 1` pods.

`maxUnavailable` define el numero maximo de pods que pueden estar no disponibles durante la actualizacion. Puede ser un numero absoluto o un porcentaje (redondeado hacia abajo). Con `maxUnavailable: 0`, nunca hay menos pods disponibles que el numero de replicas deseado, aunque durante el rollout conviven pods de la version antigua y la nueva.

**Recreate**: elimina todos los pods existentes antes de crear los nuevos. La aplicacion queda indisponible durante el proceso. Solo debe usarse cuando la aplicacion no soporta multiples versiones ejecutandose simultaneamente (por ejemplo, bases de datos con esquemas incompatibles).

```yaml
spec:
  strategy:
    type: Recreate
```

## Actualizacion de imagen

El metodo mas comun para actualizar un Deployment es cambiar la imagen del contenedor. Esto puede hacerse de forma imperativa o declarativa.

```bash
# Forma imperativa
kubectl set image deployment/api-deployment api=mi-api:3.0

# Forma declarativa (modificar el YAML y reaplicar)
kubectl apply -f deployment.yaml  # con image: mi-api:3.0

# Ver el progreso del rollout
kubectl rollout status deployment/api-deployment

# Ver el historial de revisiones
kubectl rollout history deployment/api-deployment
```

Cada vez que se modifica el template del pod (imagen, variables de entorno, recursos, comandos), el Deployment crea un nuevo ReplicaSet con la nueva configuracion y escala gradualmente el nuevo mientras reduce el anterior. Este mecanismo permite rastrear cada cambio como una revision independiente.

```bash
# Ver revisiones con detalle
kubectl rollout history deployment/api-deployment --revision=2
```

## Rollback

Si una actualizacion introduce un fallo, Kubernetes permite revertir al estado anterior con un unico comando.

```bash
# Revertir a la revision anterior
kubectl rollout undo deployment/api-deployment

# Revertir a una revision especifica
kubectl rollout undo deployment/api-deployment --to-revision=2
```

El rollout undo funciona restaurando el ReplicaSet de la revision objetivo y escalandolo al numero deseado de replicas, mientras reduce el ReplicaSet actual. Es el mismo mecanismo que una actualizacion normal, pero en sentido inverso.

Kubernetes conserva el historial de revisiones del Deployment. El numero de revisiones retenidas se controla con el campo `spec.revisionHistoryLimit` (por defecto 10). Cuando se supera ese limite, las revisiones mas antiguas se eliminan.

```yaml
spec:
  revisionHistoryLimit: 5
```

> El rollback solo funciona si el Deployment tiene historial de revisiones. Si se elimina el ReplicaSet de una revision anterior manualmente, esa revision ya no estara disponible para revertir. Nunca borres ReplicaSets a menos que estes seguro de que no los necesitaras.

## Pausar y reanudar despliegues

Kubernetes permite pausar un Deployment durante un rollout. Esto es util para realizar despliegues canary o blue-green controlados manualmente.

```bash
# Pausar el deployment
kubectl rollout pause deployment/api-deployment

# Los pods se actualizan hasta el punto de pausa y se detienen
# Se puede verificar el estado de los nuevos pods

# Reanudar el rollout
kubectl rollout resume deployment/api-deployment
```

Mientras un Deployment esta pausado, cualquier cambio en su especificacion no desencadena un nuevo rollout. Los cambios se acumulan y se aplican cuando se reanuda el Deployment.

## Comandos utiles para Deployments

```bash
# Crear un deployment directamente (imperativo, sin YAML)
kubectl create deployment api --image=mi-api:1.0 --replicas=3

# Exponer el deployment como servicio
kubectl expose deployment api --port=80 --target-port=8080 --type=ClusterIP

# Ver detalles del deployment
kubectl describe deployment api-deployment

# Ver los pods gestionados por un deployment
kubectl get pods -l app=api

# Ver en formato wide (IPs, nodos)
kubectl get pods -l app=api -o wide

# Eliminar un deployment (tambien elimina sus ReplicaSets y pods)
kubectl delete deployment api-deployment
```

> `kubectl create deployment` es util para entornos de desarrollo o testing, pero en produccion siempre debe usarse un manifiesto YAML versionado. La forma imperativa no permite especificar health probes, recursos, montar volumenes, ni definir estrategias de actualizacion.

