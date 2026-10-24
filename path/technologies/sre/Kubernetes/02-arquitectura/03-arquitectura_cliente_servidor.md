# Arquitectura cliente-servidor

Kubernetes sigue un modelo arquitectonico cliente-servidor clasico, pero con particularidades que lo hacen robusto, seguro y escalable. El API server actua como servidor central, y todos los demas componentes (kubectl, controladores, scheduler, kubelet) actuan como clientes que se comunican con el exclusivamente a traves de HTTPS. Ningun componente se comunica directamente con otro sin pasar por el API server, lo que convierte a este en el unico punto de acoplamiento y, por tanto, en el elemento critico del sistema.

Esta centralizacion no es un cuello de botella, sino una decision de diseno deliberada. Al obligar a que toda la comunicacion pase por un unico punto, Kubernetes garantiza que todas las operaciones se autentiquen, autoricen y registren. No existen canales laterales ni operaciones off-the-record.

## Flujo de operaciones

Una operacion tipica en Kubernetes, como crear un Deployment, desencadena una secuencia de eventos que involucra a todos los componentes del cluster. Comprender este flujo es fundamental para diagnosticar problemas y optimizar el rendimiento.

```bash
# El punto de partida: kubectl envía una peticion al API server
kubectl apply -f deployment.yaml --v=8
```

El flujo completo desde que el usuario ejecuta `kubectl apply` hasta que los contenedores estan corriendo es el siguiente:

1. **kubectl** lee el archivo `deployment.yaml`, lo serializa a JSON y envia una peticion HTTP POST al API server en el endpoint `/apis/apps/v1/namespaces/default/deployments`. La peticion viaja por HTTPS y kubectl presenta su certificado TLS para autenticacion.

2. **API server** recibe la peticion y la pasa por el pipeline de admision: autenticacion (verifica el certificado o token), autorizacion (consulta RBAC para ver si el usuario tiene permiso para crear Deployments), y admission controllers (plugins como `MutatingAdmissionWebhook` y `ValidatingAdmissionWebhook` que pueden modificar o rechazar la peticion).

3. **API server** serializa el objeto Deployment y lo persiste en **etcd** mediante una operacion de escritura. Si el write a etcd tiene exito, el API server responde a kubectl con un codigo 201 (Created).

4. **Controller Manager** tiene un controlador de Deployments que observa cambios en recursos del tipo Deployment via el mecanismo de watches del API server. Detecta el nuevo Deployment y crea un ReplicaSet asociado. El controlador de ReplicaSets detecta el nuevo ReplicaSet y crea los Pods especificados en la plantilla.

5. Los nuevos Pods aparecen en el API server con `spec.nodeName` vacio. **Scheduler** los detecta mediante watches y ejecuta el algoritmo de filtrado y puntuacion para seleccionar el nodo mas adecuado para cada pod. El scheduler escribe el nombre del nodo en `spec.nodeName` del pod mediante un update al API server.

6. **Kubelet** del nodo seleccionado esta observando el API server en busca de pods asignados a su nodo. Detecta el nuevo pod, consulta su spec y comienza la ejecucion: descarga la imagen del contenedor desde el registro, crea el contenedor a traves del **container runtime** (containerd, CRI-O), monta los volumenes configurados y configura la red del pod.

7. Una vez que el contenedor esta corriendo, el **kubelet** ejecuta las probes de salud configuradas (liveness, readiness, startup) y reporta periodicamente el estado del pod al API server mediante updates en el status del pod.

8. **kube-proxy** detecta los nuevos endpoints (las IPs de los pods creados) mediante watches del API server y actualiza las reglas de iptables o ipvs en el nodo para que el trafico dirigido al Service correspondiente llegue a los nuevos pods.

```
kubectl -> API Server -> etcd -> Controller Manager -> Scheduler -> Kubelet -> Container Runtime
                          ^                                                            |
                          |____________________________________________________________|
                                      (kubelet reporta estado al API server)
```

> Este flujo es el mismo para cualquier operacion, ya sea crear, actualizar o eliminar recursos. La unica diferencia es que las eliminaciones no pasan por el scheduler, y las actualizaciones pueden involucrar logica adicional en los admission controllers.

## Comunicacion HTTPS/TLS

Toda la comunicacion en Kubernetes utiliza TLS. No existen endpoints HTTP planos, ni comunicaciones internas sin cifrar. Cada componente tiene su propio par de certificados y claves, y las conexiones se autentican mutuamente (mTLS) en la mayoria de los casos.

La infraestructura PKI de Kubernetes se basa en una autoridad de certificacion (CA) interna que firma todos los certificados del cluster. Cuando se instala Kubernetes, se genera una CA raiz que luego firma certificados para el API server, el kubelet, el scheduler, el controller manager y los clientes (kubectl). Cada certificado incluye el nombre del componente y los SANs (Subject Alternative Names) necesarios para la verificacion del nombre del host.

```bash
# Los certificados del cluster se almacenan en /etc/kubernetes/pki/
ls /etc/kubernetes/pki/
# apiserver.crt         apiserver.key
# apiserver-kubelet-client.crt  apiserver-kubelet-client.key
# ca.crt                ca.key
# front-proxy-ca.crt    front-proxy-ca.key
# front-proxy-client.crt  front-proxy-client.key
# sa.pub                sa.key
```

La autenticacion entre el API server y el kubelet es especialmente relevante. El API server se conecta al kubelet para ejecutar comandos como `kubectl exec`, `kubectl logs` y `kubectl port-forward`. Esta conexion tambien va cifrada: el API server presenta su certificado de cliente (`apiserver-kubelet-client.crt`) y el kubelet verifica que este firmado por la CA del cluster.

```yaml
# Configuracion del kubelet para TLS
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
tlsCertFile: /etc/kubernetes/pki/kubelet.crt
tlsPrivateKeyFile: /etc/kubernetes/pki/kubelet.key
authentication:
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
```

Los tokens de servicio tambien se utilizan para autenticacion. Cada pod monta automaticamente un token de cuenta de servicio (ServiceAccount) en `/var/run/secrets/kubernetes.io/serviceaccount/token`. Este token permite que las aplicaciones dentro del pod se comuniquen con el API server sin necesidad de certificados adicionales.

> En entornos productivos, los certificados del cluster tienen una validez limitada (tipicamente 1 ano). La rotacion de certificados es un proceso operativo critico que debe planificarse con antelacion. Herramientas como `kubeadm` facilitan la renovacion con `kubeadm certs renew`.

## Alta disponibilidad multi-master

Un cluster con un unico plano de control es un punto unico de fallo. Si el nodo del API server cae, etcd se vuelve inaccesible, y el cluster entra en modo de solo lectura: los pods existentes continuan funcionando, pero no se pueden crear, actualizar ni eliminar recursos hasta que el API server se recupere.

La alta disponibilidad (HA) del plano de control se consigue replicando todos los componentes: multiples instancias del API server, multiples nodos de etcd formando un cluster Raft, y multiples instancias del scheduler y controller manager compitiendo por el liderazgo mediante eleccion.

```bash
# Arquitectura HA tipica con 3 nodos de control plane
# - 3 instancias de kube-apiserver detras de un balanceador de carga
# - 3 instancias de etcd en cluster (consenso Raft)
# - 3 instancias de kube-scheduler (solo una lider activo)
# - 3 instancias de kube-controller-manager (solo una lider activo)
```

El API server es stateless, por lo que escalarlo horizontalmente es sencillo: se ejecutan N instancias detras de un balanceador de carga (HAProxy, NGINX, un balanceador cloud) que distribuye las peticiones entre ellas. Todas las instancias leen y escriben en el mismo cluster de etcd, y como el API server no mantiene estado propio, no hay problemas de consistencia entre instancias.

etcd requiere configuracion especial para HA. Los nodos de etcd se configuran en cluster con direcciones conocidas entre si. El algoritmo Raft garantiza que mientras haya un quorum (mayoria de nodos disponibles), el cluster de etcd sigue funcionando. Con 3 nodos, etcd tolera la caida de 1. Con 5 nodos, tolera la caida de 2.

```bash
# Configuracion de etcd en modo cluster
# Cada nodo conoce a los otros miembros
--initial-cluster etcd-0=https://10.0.0.1:2380,etcd-1=https://10.0.0.2:2380,etcd-2=https://10.0.0.3:2380
--initial-cluster-state new
```

El scheduler y el controller manager no son stateless. Solo una instancia de cada uno debe estar activa en un momento dado para evitar operaciones duplicadas. Ambos implementan eleccion de lider usando endpoints de Kubernetes: varias instancias se ejecutan, pero solo la que consigue crear un lease (recurso Lease en el namespace kube-system) actua como lider. Las demas permanecen en espera, observando el lease y listas para asumir si el lider deja de renovarlo.

```bash
# Ver el lider actual del controller manager
kubectl get lease -n kube-system kube-controller-manager -o yaml
# spec.holderIdentity indica que instancia es la lider
# spec.leaseDurationSeconds: 15
# spec.renewTime: "2026-07-28T10:00:00Z"
```

```yaml
# Arquitectura HA completa (3 nodos de control, 3 de etcd)
# Balanceador externo (HAProxy / cloud LB)
#     |
#     +-- kube-apiserver-0  kube-apiserver-1  kube-apiserver-2
#     |         |                  |                  |
#     +--------+------------------+------------------+
#              |                  |                  |
#          etcd-0             etcd-1             etcd-2
#           (lider)           (seguidor)         (seguidor)
#
# scheduler y controller manager con eleccion de lider
# kube-scheduler-0 (lider),  kube-scheduler-1 (espera), kube-scheduler-2 (espera)
# kube-c-m-0 (lider),        kube-c-m-1 (espera),       kube-c-m-2 (espera)
```

> La alta disponibilidad del plano de control no protege contra fallos de los nodos worker. Para eso se necesita redundancia a nivel de aplicacion: multiples replicas de pods distribuidas en diferentes nodos, con PodAntiAffinity para evitar que todas las replicas caigan en el mismo worker.

## Consideraciones de red

En una arquitectura HA, el balanceador de carga que distribuye el trafico entre las instancias del API server debe configurarse para health checks en el endpoint `/healthz` del API server (puerto 6443). Si una instancia del API server falla, el balanceador la retira del pool automaticamente.

La latencia de red entre los nodos del plano de control es critica. etcd es especialmente sensible a la latencia de red: si la latencia entre nodos de etcd supera los 100 ms, el rendimiento del cluster se degrada significativamente. Por eso los nodos del plano de control deben estar en la misma zona de disponibilidad o, como minimo, con conectividad de baja latencia garantizada.

```bash
# Health check del API server para el balanceador
curl -k https://192.168.1.10:6443/healthz
# Debe responder "ok" con codigo 200

# Medir latencia entre nodos etcd
ping etcd-1
```

Los nodos worker se conectan al API server a traves de la misma direccion del balanceador de carga. En el kubelet, la configuracion `kubeconfig` apunta al endpoint del balanceador, no a una IP concreta del API server. Esto permite que los workers sigan funcionando aunque una instancia del API server caiga.

