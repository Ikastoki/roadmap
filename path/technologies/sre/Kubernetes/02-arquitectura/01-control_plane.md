# Plano de control

El plano de control (control plane) es el cerebro del cluster. Alberga los componentes que toman decisiones globales: validan y procesan cada peticion, mantienen el estado consistente del sistema, asignan cargas de trabajo a los nodos y ejecutan bucles de control que garantizan que el estado real coincida con el estado deseado. A diferencia de los nodos worker, que pueden fallar sin que el cluster deje de funcionar, una caida del plano de control bloquea la capacidad de realizar cambios, aunque las cargas existentes continuan ejecutandose.

Cada componente del plano de control tiene una responsabilidad unica y bien definida. Comprender que hace cada uno, como se comunican entre si y como se garantiza su disponibilidad es esencial para operar Kubernetes en produccion.

## kube-apiserver

El API server es el unico punto de entrada al cluster. Todas las interacciones, tanto internas como externas, pasan por el: los comandos de kubectl, las peticiones de los controladores, las llamadas del scheduler, las actualizaciones de estado del kubelet. No existe ninguna operacion en Kubernetes que no atraviese el API server.

Es un servidor RESTful que expone la API de Kubernetes sobre HTTPS en el puerto 6443. Cada peticion pasa por un pipeline de admision: autenticacion (verifica la identidad del cliente mediante certificados TLS, tokens bearer o proveedores externos), autorizacion (comprueba si el cliente tiene permisos sobre el recurso solicitado mediante RBAC o ABAC), y admission controllers (plugins que pueden mutar o validar la peticion antes de persistirla). Cualquier peticion que no supere todas las fases es rechazada.

```bash
# El API server expone sus endpoints en /api/v1, /apis/apps/v1, etc.
# Se puede consultar directamente con curl autenticado
kubectl get --raw /api/v1 | jq .resources[].name

# Verificar la version de la API
kubectl version --short
```

El API server es stateless: no almacena estado propio. Toda la informacion del cluster se persiste en etcd, y el API server actua como interfaz entre los clientes y la base de datos. Esta separacion permite escalar el API server horizontalmente: se pueden ejecutar varias instancias detras de un balanceador de carga sin problemas de consistencia.

> El API server es el unico componente que se comunica con etcd. Ningun otro componente del cluster accede directamente a la base de datos. Esto centraliza el control de acceso y garantiza que toda escritura pase por las validaciones de admision.

## etcd

etcd es la base de datos distribuida donde Kubernetes almacena todo su estado: la configuracion del cluster, los manifiestos de los recursos, el estado de cada objeto, los eventos, los secrets. Sin etcd no hay cluster: si etcd se corrompe o se pierde, Kubernetes no tiene manera de recuperar su estado.

Internamente, etcd es un almacenamiento clave-valor consistente y altamente disponible que utiliza el algoritmo de consenso Raft. Raft garantiza que, aunque algunos nodos fallen, el sistema en su conjunto mantenga la consistencia. Para que Raft funcione correctamente, etcd requiere un numero impar de nodos (tipicamente 3, 5 o 7). Con 3 nodos, soporta la caida de 1; con 5, soporta la caida de 2.

```bash
# etcdctl es la CLI para interactuar con etcd
# Listar todas las claves de Kubernetes almacenadas
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -20
```

La estructura interna de etcd sigue una jerarquia de directorios. Las claves de Kubernetes comienzan con `/registry/` seguido del tipo de recurso y su identificador: `/registry/pods/default/api-pod-xyz`, `/registry/deployments/produccion/mi-app`, `/registry/secrets/kube-system/certs`. Los valores son representaciones serializadas (protobuf) de los objetos.

etcd no esta disenado para almacenar grandes volumenes de datos. Su rendimiento se degrada con claves muy grandes o con un numero excesivo de operaciones de escritura. Por eso Kubernetes limita el tamano de los objetos que se pueden almacenar (tipicamente 1.5 MB por recurso) y recomienda no usar Secrets para almacenar archivos grandes.

> En produccion, etcd debe ejecutarse en discos SSD con baja latencia y redes dedicadas. Una latencia alta en las operaciones de disco de etcd afecta directamente a la capacidad de respuesta del API server y, por tanto, de todo el cluster.

## kube-scheduler

El scheduler es el componente encargado de asignar pods recien creados a nodos del cluster. Cuando un Deployment crea un nuevo pod, el pod aparece en el API server con `spec.nodeName` vacio. El scheduler detecta estos pods sin asignar, evalua todos los nodos disponibles y selecciona el mas adecuado para ejecutarlo.

El proceso de scheduling se divide en dos fases. La primera es el *filtering*: el scheduler descarta nodos que no cumplen los requisitos minimos del pod. Si el pod solicita 4 GB de RAM y un nodo solo tiene 2 GB disponibles, ese nodo se filtra. Si el pod tiene un `nodeSelector` que exige `disk=ssd`, los nodos sin esa etiqueta se descartan. Los taints y tolerations tambien se evaluan en esta fase.

La segunda fase es el *scoring*: el scheduler asigna una puntuacion a cada nodo que paso el filtro. Las reglas de puntuacion consideran recursos disponibles (CPU, memoria), afinidad entre pods, localidad de datos, y politicas de distribucion como `PodTopologySpread`. El nodo con mayor puntuacion recibe el pod.

```yaml
# Ejemplo de afinidad entre pods: el frontend debe ejecutarse
# en el mismo nodo que el cache
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - redis-cache
          topologyKey: kubernetes.io/hostname
```

El scheduler es extensible mediante el *scheduling framework*. Se pueden implementar plugins personalizados para las fases de filtering, scoring, reserva de recursos y post-scheduling. Esta extension permitio la creacion de schedulers especializados para cargas de trabajo de machine learning, batch processing o GPU computing sin modificar el codigo de Kubernetes.

> El scheduler no ejecuta el pod ni lo monitorea. Simplemente decide que nodo debe ejecutarlo y escribe esa decision en el API server. El kubelet del nodo seleccionado es el que realmente descarga la imagen y arranca los contenedores.

## kube-controller-manager

El controller manager es un conjunto de bucles de control (control loops) empaquetados en un unico proceso. Cada controlador observa el estado actual del cluster a traves del API server y lo compara con el estado deseado. Cuando detecta una discrepancia, ejecuta acciones correctivas.

No existe un unico controlador, sino decenas trabajando simultaneamente. El controlador de Deployments se asegura de que el numero de replicas coincida con lo declarado. El controlador de ReplicaSets crea o elimina pods para ajustarse al tamano deseado. El controlador de Nodes detecta nodos que no responden y marca sus pods para reprogramacion. El controlador de Endpoints actualiza los endpoints de los Services cuando los pods cambian. El controlador de Namespaces elimina todos los recursos dentro de un namespace cuando este se borra.

```bash
# Flags del controller manager para habilitar/deshabilitar controladores
kube-controller-manager --controllers=*,-namespace-controller

# Los controladores generan eventos que se pueden inspeccionar
kubectl get events --watch
```

Cada controlador sigue el mismo patron: un informer observa cambios en los recursos del API server, un workqueue almacena los objetos pendientes de procesar, y un worker loop ejecuta la logica de reconciliacion. Este diseno garantiza que los controladores no pierdan eventos aunque se reinicien, y que puedan procesar miles de objetos concurrentemente.

> Los controladores son el corazon del modelo declarativo de Kubernetes. Sin ellos, el cluster se limitaria a almacenar objetos en etcd sin ejecutar ninguna accion. Son los controladores los que convierten las intenciones declaradas en operaciones reales.

## cloud-controller-manager

El cloud controller manager es un componente opcional que actua como puente entre Kubernetes y el proveedor de nube. Ejecuta controladores que dependen de las APIs del proveedor cloud: gestion de balanceadores de carga (Services tipo LoadBalancer), rutas de red, volumenes persistentes y deteccion de fallos en nodos.

Su existencia responde a un principio de diseno fundamental: Kubernetes no debe depender de ningun proveedor cloud concreto. Separando la logica cloud en un componente independiente, el nucleo de Kubernetes permanece agnostico del entorno. Cada proveedor (AWS, GCP, Azure, DigitalOcean) implementa su propio cloud-controller-manager.

```yaml
# Cuando se crea un Service tipo LoadBalancer
# cloud-controller-manager llama a la API del cloud provider
# para provisionar un balanceador de carga externo
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```

En clusters on-premise o en entornos que no requieren integracion cloud, este componente no se despliega. Alternativas como MetalLB, el balanceador de carga para bare metal, sustituyen la funcionalidad del cloud-controller-manager mediante sus propios controladores.

