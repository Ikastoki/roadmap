# Creacion de Pods

Un pod es la unidad minima de computo en Kubernetes. No los contenedores de forma individual, sino los pods. Cada pod encapsula uno o varios contenedores que comparten la misma direccion IP, el mismo espacio de nombres de red y el mismo ciclo de vida. Cuando un pod se crea, todos sus contenedores arrancan juntos en el mismo nodo. Cuando se destruye, todos mueren. Esta atomicidad es la base sobre la que se construye el resto del modelo de objetos de Kubernetes.

Aunque los pods son la unidad basica de ejecucion, rara vez se crean directamente en produccion. Son efimeros por diseno: no se recuperan automaticamente, no se reemplazan si el nodo falla, y no tienen capacidad de escalado. Para eso existen los controladores como Deployments o StatefulSets. Sin embargo, entender la estructura de un pod es imprescindible, porque es el bloque fundamental que todos esos controladores gestionan.

## Creacion imperativa con kubectl run

El metodo mas rapido para crear un pod es mediante el comando `kubectl run`. Este comando crea un pod directamente a partir de una imagen de contenedor, sin necesidad de escribir un archivo YAML.

```bash
kubectl run nginx --image=nginx:1.25

# Ver el pod creado
kubectl get pods

# Ver detalles del pod
kubectl describe pod nginx
```

El pod se crea en el namespace activo (por defecto `default`) y se le asigna una IP interna del cluster. El contenedor se descarga y ejecuta automaticamente. Si la imagen no existe localmente, el kubelet la descarga del registro configurado (Docker Hub por defecto).

`kubectl run` acepta multiples opciones para personalizar el pod: variables de entorno, puertos expuestos, etiquetas, limites de recursos, comandos de entrada, y montar ConfigMaps o Secrets. Sin embargo, esta forma imperativa no es recomendable para entornos de produccion porque no deja rastro del estado deseado mas alla del recurso vivo en el cluster.

```bash
# Crear un pod con opciones adicionales
kubectl run api \
  --image=mi-api:1.0 \
  --port=8080 \
  --env="DB_HOST=postgres.prod.svc.cluster.local" \
  --labels="tier=backend,env=prod" \
  --requests="cpu=250m,memory=256Mi" \
  --limits="cpu=500m,memory=512Mi"
```

## Creacion declarativa con YAML

El enfoque declarativo consiste en escribir un manifiesto YAML que describe el estado deseado del pod y aplicarlo con `kubectl apply`. Este archivo es la fuente de verdad y puede versionarse en Git, revisarse en code review, y reutilizarse en distintos entornos.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
    tier: frontend
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
          protocol: TCP
      env:
        - name: ENVIRONMENT
          value: production
```

```bash
# Aplicar el manifiesto
kubectl apply -f pod.yaml

# Ver el pod en formato YAML (como lo almacena Kubernetes)
kubectl get pod web-pod -o yaml
```

La estructura basica de un manifiesto de pod tiene cuatro campos esenciales:

**apiVersion**: especifica la version de la API de Kubernetes que se utiliza. Para pods es siempre `v1`. Cada recurso tiene su propia version de API, y estas versiones cambian a medida que Kubernetes evoluciona (`v1`, `v1beta1`, `v1alpha1`).

**kind**: el tipo de recurso que se esta definiendo. En este caso `Pod`. Cada objeto de Kubernetes tiene un kind unico que lo identifica: `Deployment`, `Service`, `ConfigMap`, `Ingress`, etc.

**metadata**: informacion que identifica al recurso: `name` (obligatorio, unico dentro del namespace), `labels` (pares clave-valor para organizar y seleccionar recursos), `annotations` (metadatos no identificativos, como informacion de herramientas externas), y `namespace` (opcional, si no se especifica se usa el namespace del contexto actual).

**spec**: la especificacion del estado deseado. Para un pod, el campo mas importante es `containers`, que es una lista de contenedores que se ejecutaran dentro del pod. Cada contenedor requiere al menos un `name` y una `image`. Opcionalmente incluye `ports`, `env`, `resources`, `volumeMounts`, `command`, `args`, y los health probes.

## Labels y selectors

Las labels son el mecanismo de organizacion fundamental en Kubernetes. Cada recurso puede tener multiples labels, y los controladores utilizan selectores para identificar que pods gestionar.

```yaml
metadata:
  labels:
    app: web
    environment: production
    release: v2.0
    tier: frontend
    managed-by: terraform
```

Los selectores permiten filtrar recursos por labels. El selector mas basico es `matchLabels`, que selecciona recursos que tienen todas las labels especificadas. Existe tambien `matchExpressions` para condiciones mas complejas (In, NotIn, Exists, DoesNotExist).

```bash
# Filtrar pods por label
kubectl get pods -l app=web
kubectl get pods -l "app=web,environment=production"
kubectl get pods -l "tier in (frontend,backend)"
```

## Ciclo de vida de un pod

Un pod atraviesa varias fases desde que se crea hasta que se destruye. Kubernetes expone estas fases en el campo `status.phase` y son visibles con `kubectl get pods`.

**Pending**: el pod ha sido aceptado por el cluster pero aun no se ha ejecutado ningun contenedor. Puede estar descargando la imagen, esperando que el scheduler asigne un nodo, o esperando que se monten volumenes.

**Running**: al menos un contenedor del pod se esta ejecutando. El pod permanece en esta fase hasta que sus contenedores terminan o se decide destruirlo.

**Succeeded**: todos los contenedores del pod han terminado con codigo de salida 0 (exito). Esta fase es normal para pods que ejecutan trabajos batch (Jobs, CronJobs).

**Failed**: al menos un contenedor ha terminado con codigo de salida distinto de cero.

**Unknown**: el kubelet no ha podido comunicar el estado del pod al API server. Generalmente indica un problema de red o que el nodo ha fallado.

Ademas de la fase, cada contenedor dentro del pod tiene condiciones internas que Kubernetes monitoriza:

```bash
# Ver condiciones del pod
kubectl get pod web-pod -o jsonpath='{.status.conditions}'

# Ver el estado detallado con eventos
kubectl describe pod web-pod
# Los eventos aparecen al final de la salida e incluyen:
#   Scheduled - El pod ha sido asignado a un nodo
#   Pulling - Descargando la imagen
#   Pulled - Imagen descargada
#   Created - Contenedor creado
#   Started - Contenedor iniciado
```

## Pods multi-contenedor

Un mismo pod puede alojar varios contenedores que comparten la misma IP, el mismo espacio de puertos y los mismos volumenes. Los contenedores se comunican entre si mediante localhost y comparten el ciclo de vida: si uno falla y se reinicia, todos se reinician.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
    - name: sidecar
      image: curlimages/curl
      command: ["sh", "-c", "while true; do curl -s http://localhost:80; sleep 5; done"]
```

Este patron, conocido como sidecar, es una de las estrategias mas potentes de Kubernetes. El contenedor principal ejecuta la logica de negocio, mientras que el sidecar proporciona funcionalidades auxiliares: un proxy de red (Istio, Envoy), un recolector de logs (Fluentd, Filebeat), o un adaptador de metricas (Prometheus exporter). Ambos contenedores comparten red y almacenamiento, eliminando la sobrecarga de configurar comunicacion externa entre ellos.

## Comandos y argumentos

Kubernetes permite sobrescribir el punto de entrada y los argumentos del contenedor definidos en la imagen. Los campos `command` y `args` en la especificacion del contenedor reemplazan `ENTRYPOINT` y `CMD` del Dockerfile respectivamente.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-pod
spec:
  containers:
    - name: alpine
      image: alpine:3.18
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo 'Hola desde Kubernetes'; sleep 5; done"]
```

```bash
# Ver logs del pod para confirmar la salida
kubectl logs command-pod
```

## Variables de entorno

Las variables de entorno se definen en el campo `env` de cada contenedor. Ademas de valores literales, pueden obtenerse de campos del pod (fieldRef), de recursos del namespace (resourceFieldRef), de ConfigMaps (configMapKeyRef), o de Secrets (secretKeyRef).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
    - name: app
      image: mi-app
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: DB_URL
          value: "postgres://user:pass@db-service:5432/mydb"
```

> Las variables de entorno se inyectan en el contenedor antes de que el proceso principal arranque. Si un ConfigMap o Secret del que depende una variable no existe, el contenedor no se inicia hasta que el recurso este disponible. Esto previene arranques con configuracion incompleta.

## Recursos y limites

Kubernetes permite especificar los recursos de CPU y memoria que cada contenedor necesita y que puede consumir como maximo. El scheduler utiliza `requests` para decidir en que nodo colocar el pod, y `limits` para evitar que un contenedor consuma todos los recursos del nodo.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resources-pod
spec:
  containers:
    - name: app
      image: mi-app
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

Un contenedor que supera el limite de CPU se estrangula (throttling). Un contenedor que supera el limite de memoria se mata (OOMKilled). Los contenedores que no especifican recursos pueden consumir toda la capacidad del nodo, degradando el rendimiento de otros pods.

> La unidad `m` para CPU representa milicores: 1000m equivale a 1 nucleo completo. Para memoria, `Mi` son mebibytes (2^20 bytes) y `Gi` son gibibytes (2^30 bytes). Nunca uses `MB` o `GB` (decimal) en Kubernetes a menos que tengas una razon muy especifica.

