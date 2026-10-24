# Conceptos clave

Kubernetes introduce un modelo de objetos que abstrae la infraestructura fisica y expone al administrador una API declarativa para describir el estado deseado del sistema. Cada objeto representa una intencion: un numero de replicas, una configuracion de red, un volumen de almacenamiento, un conjunto de credenciales. Comprender estos objetos es comprender Kubernetes.

Estos conceptos no son piezas aisladas. Se combinan y relacionan formando el modelo de aplicacion de Kubernetes. Un Deployment gestiona Pods, un Pod monta Volumes y consume ConfigMaps, un Service expone esos Pods, un Ingress enruta trafico desde fuera del cluster hacia el Service. La mayoria de los problemas operativos con Kubernetes se reducen a no entender como se conectan estas piezas.

## Cluster

Un cluster es el conjunto completo de maquinas (fisicas o virtuales) sobre las que Kubernetes orquesta contenedores. Esta compuesto por dos grupos de nodos: los del plano de control y los de trabajo. El cluster es la unidad atomica de Kubernetes: no tiene sentido hablar de un despliegue "en Kubernetes" sin especificar a que cluster pertenece.

La mayoria de las interacciones con Kubernetes se realizan a traves de la API del cluster. El kubeconfig es el archivo de configuracion que contiene los datos de autenticacion y conexion a uno o varios clusters. Cada entrada en kubeconfig define un cluster, un usuario (credenciales) y un contexto que los asocia.

```bash
# Ver contextos configurados
kubectl config get-contexts

# Cambiar de contexto (cluster)
kubectl config use-context produccion

# Informacion del cluster
kubectl cluster-info
```

## Nodo

Un nodo es una maquina dentro del cluster. Puede ser un servidor fisico, una maquina virtual en la nube o un contenedor en un entorno de desarrollo local (como Kind o Minikube). Cada nodo ejecuta tres componentes esenciales: el kubelet (agente que se comunica con el plano de control), el kube-proxy (encargado de las reglas de red) y el runtime de contenedores (containerd, CRI-O, etc.).

Los nodos no son gestionados directamente por el administrador. Kubernetes decide que nodo ejecuta cada pod basandose en restricciones de recursos, afinidad, taints y tolerations. Cuando un nodo falla o deja de responder, Kubernetes lo marca como `NotReady` y reprograma sus pods en otros nodos.

```bash
# Listar nodos del cluster
kubectl get nodes

# Ver detalles de un nodo (capacidad, recursos, condiciones)
kubectl describe node nodo-1
```

## Pod

El pod es la unidad minima de computo en Kubernetes. No los contenedores de forma individual, sino los pods. Un pod encapsula uno o varios contenedores que comparten la misma direccion IP, el mismo espacio de nombres de red y el mismo volumen de almacenamiento. Los contenedores dentro de un pod se comunican entre si mediante localhost y comparten el ciclo de vida: se crean y destruyen juntos.

El caso mas comun es un pod con un unico contenedor, pero el patron sidecar coloca un contenedor adicional en el mismo pod para ampliar funcionalidades: un proxy de red (Envoy, Istio), un recolector de logs (Fluentd), o un adaptador de metricas. Ambos contenedores comparten red y almacenamiento, eliminando la necesidad de configurar comunicacion entre ellos.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

> Los pods son efimeros por diseno. No se crean pods directamente en produccion; se crean a traves de controladores como Deployments o StatefulSets que gestionan su ciclo de vida.

## Deployment

Un Deployment es un controlador que gestiona el estado deseado de un conjunto de pods. Declaras cuantas replicas quieres, que imagen de contenedor deben usar, y Kubernetes mantiene ese estado en el tiempo. Si un pod falla, el Deployment lo recrea. Si actualizas la imagen, el Deployment realiza un rolling update.

El Deployment es el recurso mas utilizado para aplicaciones stateless. Define la estrategia de actualizacion (rolling, recreate), el numero de replicas, los health checks, y las politicas de escalado. Detras del Deployment opera un ReplicaSet, que es el responsable directo de mantener el numero de pods en ejecucion.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
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
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
```

## Service

Los pods son efimeros y sus direcciones IP cambian cada vez que se recrean. Un Service es una abstraccion que define un conjunto logico de pods y una politica para acceder a ellos. Kubernetes asigna al Service una IP virtual estable y un nombre DNS que permanecen constantes independientemente de los pods subyacentes.

Existen varios tipos de Service. `ClusterIP` expone el servicio dentro del cluster, `NodePort` lo expone en un puerto estatico de cada nodo, `LoadBalancer` provisiona un balanceador de carga en el proveedor cloud, y `ExternalName` devuelve un registro CNAME.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

## Ingress

Un Service LoadBalancer expone un servicio en cada puerto, pero si tienes varios servicios (api, frontend, admin), necesitas multiples balanceadores de carga, cada uno con su propia IP y costo asociado. Ingress resuelve esto proporcionando enrutamiento HTTP/HTTPS a nivel de aplicacion: un unico punto de entrada que distribuye las peticiones a diferentes servicios segun el hostname o la ruta.

Ingress no funciona por si solo. Necesita un controlador de Ingress instalado en el cluster, como NGINX Ingress Controller, Traefik, HAProxy o el controlador nativo de AWS (ALB Ingress Controller). El recurso Ingress define las reglas de enrutamiento, y el controlador las implementa configurando el proxy correspondiente.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: frontend.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

## ConfigMap y Secret

ConfigMap y Secret son los mecanismos de Kubernetes para separar la configuracion de la imagen del contenedor. Un ConfigMap almacena datos de configuracion no sensibles en formato clave-valor o como archivos. Un Secret hace lo mismo pero para informacion sensible (credenciales, tokens, claves SSH), con codificacion base64 y opciones de cifrado en reposo.

Ambos recursos se inyectan en los pods como variables de entorno, como archivos montados en un volumen, o pasandolos como argumentos de linea de comandos. La separacion entre ConfigMap y Secret no es solo semantica: los Secrets tienen politicas de acceso mas restrictivas y pueden integrarse con sistemas externos de gestion de secretos como HashiCorp Vault o AWS Secrets Manager.

```yaml
# ConfigMap para configuracion no sensible
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  DB_HOST: "postgres.prod.svc.cluster.local"

# Secret para credenciales
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DB_PASSWORD: "s3cr3t-p4ssword"
  API_KEY: "sk-abc123def456"
```

> Nunca incluyas Secrets en los manifiestos que se suben al repositorio. Utiliza herramientas como SealedSecrets, Helm Secrets, External Secrets Operator o un vault provider para gestionar secretos de forma segura fuera del control de versiones.

## Namespace

Un namespace es un mecanismo de aislamiento virtual dentro de un cluster. Permite dividir un cluster fisico en multiples clusters logicos, cada uno con sus propios recursos, cuotas y politicas de acceso. Los nombres de los recursos deben ser unicos dentro de un namespace, pero pueden repetirse en distintos namespaces.

Los namespaces son especialmente utiles en entornos con multiples equipos o multiples entornos (desarrollo, staging, produccion) compartiendo el mismo cluster. Kubernetes proporciona tres namespaces por defecto: `default` (donde se crean los recursos si no se especifica otro), `kube-system` (recursos internos del sistema) y `kube-public` (recursos legibles por todos los usuarios).

```bash
# Listar namespaces
kubectl get namespaces

# Crear recursos en un namespace especifico
kubectl apply -f deployment.yaml -n produccion

# Ver pods de todos los namespaces
kubectl get pods --all-namespaces
```

## Volume

Los contenedores son efimeros: cuando un pod muere, su sistema de archivos desaparece con el. Un Volume es un directorio accesible desde el contenedor que persiste independientemente del ciclo de vida del pod. Kubernetes soporta decenas de tipos de volumen: `emptyDir` (temporal, compartido entre contenedores del mismo pod), `hostPath` (directorio del nodo), `persistentVolumeClaim` (almacenamiento persistente gestionado externamente), `configMap`, `secret`, y los especificos de cada proveedor cloud (EBS, CSI, Azure Disk, etc.).

El sistema de PersistentVolume (PV) y PersistentVolumeClaim (PVC) abstrae la gestion del almacenamiento. El administrador provisiona PVs (discos en la infraestructura), y los desarrolladores crean PVCs que solicitan almacenamiento con ciertas caracteristicas (capacidad, modo de acceso). Kubernetes empareja PVCs con PVs compatibles y los monta en los pods.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: mi-app
      volumeMounts:
        - name: data
          mountPath: /var/lib/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

## Kubelet

El kubelet es el agente que se ejecuta en cada nodo del cluster. Es el unico componente que se comunica directamente con el plano de control desde el nodo. Su funcion principal es garantizar que los contenedores definidos en los PodSpecs esten ejecutandose y sanos.

El kubelet registra el nodo en el cluster, recibe las especificaciones de los pods asignados a ese nodo desde el API server, descarga las imagenes necesarias, ejecuta los contenedores a traves del runtime, y reporta periodicamente el estado del nodo y de sus pods al plano de control. Tambien ejecuta los health checks (liveness, readiness, startup) configurados en los pods y reinicia contenedores que fallan las pruebas.

```bash
# Ver logs del kubelet
journalctl -u kubelet

# Ver el estado del kubelet
systemctl status kubelet

# El kubelet expone un endpoint de metricas en el puerto 10250
# (requiere autenticacion)
```

## kubectl

kubectl es la herramienta de linea de comandos para interactuar con la API de Kubernetes. No es un componente del cluster, sino el cliente principal que usan administradores y desarrolladores. Cada comando de kubectl se traduce en una peticion HTTP al API server, ya sea para obtener el estado de un recurso, crearlo, modificarlo o eliminarlo.

El poder de kubectl reside en su modelo declarativo. El comando `kubectl apply -f manifiesto.yaml` no dice "crea este recurso" sino "asegurate de que el estado del cluster coincida con lo que describe este archivo". Si el recurso no existe, se crea. Si existe y ha cambiado, se actualiza. Si existe pero no se incluye en la operacion, no se modifica. Este enfoque es fundamental en flujos GitOps donde los manifiestos son la fuente de verdad.

```bash
# Forma imperativa (no recomendada en produccion)
kubectl run nginx --image=nginx

# Forma declarativa (recomendada)
kubectl apply -f deployment.yaml

# Atajos comunes
kubectl get pods -o wide           # Informacion extendida (IP, nodo)
kubectl describe pod nginx-pod     # Detalles completos y eventos
kubectl logs -f deployment/api     # Logs en tiempo real de un deployment
kubectl exec -it pod-name -- sh    # Acceso a un contenedor en ejecucion
kubectl port-forward pod-name 8080:80  # Reenvio de puertos local
kubectl delete pod --all           # Eliminar todos los pods
```

```bash
# Autocompletado para kubectl (bash o zsh)
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

