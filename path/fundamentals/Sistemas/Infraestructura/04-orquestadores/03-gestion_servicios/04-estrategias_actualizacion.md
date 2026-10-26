# Estrategias de actualizacion

La forma en que se actualiza un servicio en produccion determina directamente la disponibilidad del sistema y el riesgo asociado al cambio. Un despliegue mal gestionado puede provocar interrupciones de servicio, perdida de conexiones activas o corrupcion de datos si los clientes no estan preparados para la nueva version. Los orquestadores de contenedores proporcionan mecanismos nativos para gestionar actualizaciones de forma controlada, y la eleccion de la estrategia adecuada depende de factores como la tolerancia al downtime, la capacidad de la aplicacion para ejecutarse en versiones simultaneas, el ritmo de despliegue necesario y la complejidad operativa que el equipo puede asumir.

## Rolling Update (actualizacion gradual)

La estrategia por defecto en Kubernetes es el Rolling Update, que reemplaza las instancias antiguas de forma paulatina. El orquestador crea nuevas revisiones del Pod con la version actualizada y, solo cuando estas pasan el health check de readiness, elimina las antiguas. El parametro `maxUnavailable` controla cuantas instancias antiguas pueden estar fuera de servicio simultaneamente, y `maxSurge` controla cuantas instancias nuevas pueden existir por encima del numero deseado. Con `maxUnavailable: 0` y `maxSurge: 1`, el orquestador nunca reduce la capacidad disponible: primero crea una instancia nueva, verifica que esta lista, y solo entonces elimina una antigua. Este enfoque garantiza cero downtime siempre que la aplicacion sea compatible con la coexistencia de versiones (lo que implica que los clientes no guarden estado local y que la API sea retrocompatible).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 4
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
```

## Recreate (recreacion completa)

La estrategia Recreate elimina todas las instancias antiguas antes de crear las nuevas. Esto implica un periodo de indisponibilidad inevitable, pero es necesaria en escenarios donde la coexistencia de versiones es problematica: aplicaciones que adquieren bloqueos exclusivos en bases de datos, servicios que utilizan archivos de cache compartidos con formato incompatible entre versiones, o procesos que necesitan un unico punto de entrada para evitar conflictos de concurrencia. El downtime controlado puede minimizarse programando el despliegue en ventanas de mantenimiento, pero no puede eliminarse completamente.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: migrador-db
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: migrador-db
  template:
    metadata:
      labels:
        app: migrador-db
    spec:
      containers:
        - name: migrador
          image: registry.empresa.com/migrador-db:v3.0.0
```

> La estrategia Recreate es preferible a Rolling Update cuando la aplicacion utiliza un archivo de configuracion binario que no es retrocompatible entre versiones. En Rolling Update, el nodo ejecutaria simultaneamente Pods con configuraciones incompatibles, provocando errores intermitentes difiles de diagnosticar.

## Blue-Green (despliegue azul-verde)

El patron Blue-Green mantiene dos entornos de produccion identicos: uno sirve trafico activo (azul) y el otro esta desplegado pero inactivo (verde). Cuando se realiza un cambio, se despliega la nueva version en el entorno inactivo, se verifica su correcto funcionamiento, y se redirige todo el trafico de golpe hacia el nuevo entorno. El cambio de trafico se implementa en Kubernetes modificando el Service para que apunte a los Pods de la nueva revision mediante el selector de etiquetas. Esta estrategia ofrece cero downtime durante la transicion y permite una reversión instantánea simplemente volviendo a apuntar el Service al entorno anterior, pero tiene un costo de recursos significativo: el doble de capacidad esta desplegada durante el periodo de transicion.

```yaml
# Entorno activo (azul) - desplegado actualmente
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway-blue
  labels:
    app: api-gateway
    slot: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
      slot: blue
  template:
    metadata:
      labels:
        app: api-gateway
        slot: blue
    spec:
      containers:
        - name: gateway
          image: registry.empresa.com/api-gateway:v2.3.0
---
# Entorno de preproduccion (verde) - listo para activar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway-green
  labels:
    app: api-gateway
    slot: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
      slot: green
  template:
    metadata:
      labels:
        app: api-gateway
        slot: green
    spec:
      containers:
        - name: gateway
          image: registry.empresa.com/api-gateway:v2.4.0
---
# Service que apunta al entorno activo
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-svc
spec:
  selector:
    app: api-gateway
    slot: blue   # Cambiar a "green" para activar la nueva version
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Activar el entorno verde
kubectl patch service api-gateway-svc -p '{"spec":{"selector":{"slot":"green"}}}'

# Verificar que el trafico se esta dirigiendo a la nueva revision
kubectl get endpoints api-gateway-svc -n produccion
```

## Canary (despliegue canario)

La estrategia Canary consiste en dirigir una fraccion pequeña del trafico hacia la nueva version mientras el resto sigue llegando a la version estable. Esto permite validar el comportamiento de la nueva version con trafico real antes de un despliegue completo, detectando problemas de rendimiento, errores logicos o incompatibilidades que las pruebas automaticas no capturan. En Kubernetes, el Canary se implementa manualmente mediante despliegues con un numero reducido de replicas y Services que seleccionan por etiquetas versionadas, o de forma automatizada con herramientas como Argo Rollouts, Flagger o Istio que gestionan el porcentaje de trafico de forma programatica.

```yaml
# Version estable (mayor trafico)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: api-gateway
      track: stable
  template:
    metadata:
      labels:
        app: api-gateway
        track: stable
    spec:
      containers:
        - name: gateway
          image: registry.empresa.com/api-gateway:v2.3.0
---
# Version canary (trafico reducido)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-gateway
      track: canary
  template:
    metadata:
      labels:
        app: api-gateway
        track: canary
    spec:
      containers:
        - name: gateway
          image: registry.empresa.com/api-gateway:v2.4.0
---
# Service que selecciona ambos tracks (10% canary, 90% stable)
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-svc
spec:
  selector:
    app: api-gateway    # Sin filtro de track: ambos sets de Pods reciben trafico
  ports:
    - port: 80
      targetPort: 8080
```

> El ratio de trafico en un Canary implementado con Services de Kubernetes es proporcional al numero de replicas, no un porcentaje exacto. Con 9 replicas stable y 1 canary, el trafico se distribuye aproximadamente al 90/10, pero la distribucion real depende del balanceo de carga del kube-proxy y no es perfectamente determinista. Para un control preciso del porcentaje de trafico, se requiere un service mesh como Istio o Linkerd que gestione el enrutamiento a nivel de L7.

