# Resource Limits

Un cluster Kubernetes sin limites de recursos es un sistema en anarquia. Un solo Pod con una fuga de memoria puede consumir toda la RAM de un nodo, provocando que el kubelet termine otros Pods para liberar espacio. Un bucle infinito en una aplicacion puede saturar la CPU, degradando el rendimiento de todos los contenedores del nodo. Los resource limits existen para evitar que un comportamiento anormal en un contenedor afecte al resto del cluster, garantizando que cada Pod reciba los recursos que necesita sin robarle a los demas.

## Requests: recursos garantizados

Un `request` es la cantidad de recursos que Kubernetes garantiza a un contenedor. Cuando el kubelet programa un Pod en un nodo, verifica que la suma de requests de todos los contenedores del Pod no supere la capacidad disponible del nodo. Si no hay un nodo con suficientes recursos libres, el Pod queda en estado `Pending`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-web
spec:
  containers:
    - name: web
      image: nginx
      resources:
        requests:
          cpu: 250m
          memory: 256Mi
```

CPU se mide en milicores: 250m equivale a un cuarto de nucleo, 1000m a un nucleo completo. Memoria se expresa en bytes con sufijos Ki, Mi, Gi. El request no limita el consumo maximo; si hay recursos disponibles en el nodo, el contenedor puede usar mas. El request simplemente reserva esa cantidad para que el kubelet no la asigne a otros contenedores.

El scheduler utiliza los requests para tomar decisiones de placement. Si un nodo tiene 8 Gi de RAM y ya hay Pods con requests que suman 6 Gi, solo aceptara nuevos Pods cuyo request de memoria sume 2 Gi o menos. Este mecanismo evita la sobreprogramacion excesiva, aunque el consumo real en el nodo pueda ser inferior al reservado.

```bash
# Ver recursos asignados y disponibles en los nodos
kubectl describe nodes | grep -A 5 "Allocated resources"
# Allocated resources:
#   (Total limits may be over 100 percent, i.e., overcommitted.)
#   Resource           Requests      Limits
#   cpu                1500m (37%)   2500m (62%)
#   memory             512Mi (6%)    2Gi (26%)
```

## Limits: limites maximos

Un `limit` establece el maximo de recursos que un contenedor puede consumir. Si un contenedor intenta superar su limit de CPU, el kernel lo estrangula (throttling), ralentizando sus ciclos de ejecucion. Si supera el limit de memoria, el kernel lo mata con un OOM kill (Out Of Memory).

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Cuando un contenedor es eliminado por OOM, el kubelet lo reinicia segun la politica de reinicio del Pod (`restartPolicy`). Si el OOM persiste, el contenedor entrara en un bucle de crash-reinicio que puede degradar el servicio. Por eso es importante establecer limits realistas, no solo para proteger al nodo sino para que el equipo operativo detecte problemas de memoria en la aplicacion.

> No establecer limits de memoria (`limits: {}`) es peligroso. Si un contenedor no tiene limit de memoria y sufre una fuga, puede consumir toda la RAM del nodo. El kubelet se vera obligado a terminar otros Pods para estabilizar el nodo, causando una cascade de fallos.

## Quality of Service classes

Kubernetes clasifica los Pods en tres clases de Quality of Service (QoS) segun como se definen requests y limits. Esta clasificacion determina que Pods se terminan primero cuando el nodo tiene presion de recursos.

**Guaranteed:** todos los contenedores del Pod tienen requests y limits identicos para CPU y memoria. Es la clase mas alta. Estos Pods son los ultimos en ser terminados cuando el nodo esta bajo presion.

```yaml
resources:
  requests:
    cpu: 500m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

**Burstable:** al menos un contenedor tiene requests y limits distintos (request < limit) o solo tiene requests. Es la clase mas comun en entornos de produccion. Estos Pods pueden usar recursos extras cuando el nodo tiene capacidad libre, pero seran terminados antes que los Guaranteed si hay contention.

```yaml
resources:
  requests:
    cpu: 250m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

**BestEffort:** ningun contenedor del Pod especifica requests ni limits. Es la clase mas baja. Estos Pods pueden usar cualquier recurso disponible, pero son los primeros en ser sacrificados cuando el nodo necesita liberar memoria. No deberian usarse en produccion a menos que la aplicacion sea totalmente prescindible.

```yaml
resources: {}
```

```bash
# Ver la QoS class de un Pod
kubectl get pod app-web -o jsonpath='{.status.qosClass}'
# Guaranteed, Burstable, o BestEffort
```

## Eviction y presion de recursos

El kubelet monitoriza constantemente el estado de los recursos del nodo. Cuando la memoria disponible cae por debajo de un umbral, el kubelet activa el proceso de eviction: comienza a terminar Pods empezando por los BestEffort, luego los Burstable, y por ultimo los Guaranteed. Dentro de cada clase, los Pods que mas consumen por encima de su request son los primeros en ser sacrificados.

```bash
# Ver umbrales de eviction del kubelet
kubectl describe node | grep -i eviction
# evictionHard:
#   imagefs.available: 15%
#   memory.available: 100Mi
#   nodefs.available: 10%
```

Los umbrales de eviction incluyen `memory.available`, `nodefs.available` (disco del nodo) e `imagefs.available` (disco de imagenes). Cuando se cruza un umbral, el kubelet intenta liberar recursos terminando Pods antes de que el nodo entre en un estado critico.

> Los Pods con QoS Guaranteed no son inmunes a la terminacion. Si el propio Pod Guaranteed supera su limit de memoria y el kernel lo mata con OOMKill, el Pod se reinicia. La clase QoS solo protege contra la terminacion iniciada por el kubelet por presion de recursos en el nodo, no contra fallos internos del contenedor.

## Buenas practicas con resource limits

Establecer requests y limits requiere conocer el perfil de consumo de la aplicacion. No hacerlo es negligencia; hacerlo con valores arbitrarios es igualmente peligroso. La recomendacion es medir el consumo real en desarrollo y staging antes de fijar valores en produccion.

```yaml
# Perfil tipico para un microservicio web
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

Los limits de CPU deben ser holgados para permitir picos de procesamiento sin estrangulamiento excesivo. Los limits de memoria deben ser estrictos: superar el limit de memoria mata el contenedor, pero la alternativa (dejarlo crecer sin control) es peor. Una aplicacion bien disenada debe poder reiniciarse sin perder estado ni corromper datos.

Es recomendable utilizar `LimitRange` para establecer valores por defecto a nivel de namespace. Esto evita que un equipo olvide configurar recursos y termine desplegando Pods BestEffort sin saberlo:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: desarrollo
spec:
  limits:
    - default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 200m
        memory: 256Mi
      type: Container
```

Los `ResourceQuota` permiten establecer limites agregados a nivel de namespace. Un namespace no puede consumir mas alla de lo definido en su quota, independientemente de los Pods individuales que contenga:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: desarrollo
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: 10
    pods: 20
```

La combinacion de requests, limits, LimitRange y ResourceQuota proporciona un sistema completo de gobierno de recursos. Los equipos pueden operar dentro de sus cuotas sin riesgo de afectar a otros namespaces, y el cluster entero queda protegido contra consumos descontrolados.

