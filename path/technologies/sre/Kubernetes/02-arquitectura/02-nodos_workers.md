# Nodos workers

Los nodos worker son las maquinas que realmente ejecutan las cargas de trabajo. Mientras el plano de control toma decisiones globales, los workers proporcionan la capacidad de computo, red y almacenamiento que los contenedores necesitan. Un cluster puede tener desde uno hasta miles de nodos worker, y la relacion entre el plano de control y los workers es estrictamente jerarquica: los workers nunca inician acciones por si mismos, solo responden a las instrucciones del plano de control.

Cada nodo worker ejecuta tres componentes esenciales: el kubelet, el kube-proxy y el container runtime. Los tres deben estar instalados y configurados correctamente para que el nodo sea considerado `Ready` y pueda recibir pods.

## Kubelet

El kubelet es el agente principal del nodo. Se ejecuta como un proceso del sistema (gestionado por systemd en distribuciones Linux) y actua como el puente de comunicacion entre el plano de control y el nodo worker. Es el unico componente del nodo que se comunica directamente con el API server.

Su funcion principal es garantizar que los contenedores definidos en los PodSpecs asignados a su nodo esten ejecutandose y en buen estado. El kubelet no crea pods por iniciativa propia; recibe las especificaciones de pods del API server, las interpreta y las traduce en acciones sobre el container runtime: descargar imagenes, crear contenedores, montar volúmenes, configurar redes.

El ciclo de vida de un pod desde la perspectiva del kubelet sigue estos pasos: el kubelet observa el API server en busca de pods asignados a su nodo (`spec.nodeName` coincide con el nombre del nodo). Cuando detecta un nuevo pod, consulta su spec, valida que las imagenes existen o las descarga del registro, crea los contenedores a traves del runtime, configura las probes de salud (liveness, readiness, startup) y comienza a reportar periodicamente el estado del pod y del nodo al API server.

```bash
# El kubelet expone un endpoint de metricas en /metrics
# Accesible (con autenticacion) en el puerto 10250
curl -sk https://localhost:10250/metrics \
  --cert /etc/kubernetes/pki/kubelet-client.crt \
  --key /etc/kubernetes/pki/kubelet-client.key

# Ver logs del kubelet
journalctl -u kubelet -f
```

El kubelet tambien ejecuta los health checks. La liveness probe determina si el contenedor sigue vivo; si falla, el kubelet reinicia el contenedor. La readiness probe determina si el contenedor esta listo para recibir trafico; si falla, el kubelet elimina el pod de los endpoints del Service. La startup probe se usa para contenedores que tardan en arrancar; mientras no supere esta probe, las otras no se ejecutan.

```yaml
# Configuracion de probes en un pod
spec:
  containers:
    - name: api
      image: mi-api:2.0
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 5
```

> El kubelet no gestiona contenedores que no haya creado el mismo. Si un contenedor se ejecuta en el nodo pero no forma parte de un pod gestionado por Kubernetes, el kubelet lo ignora por completo.

## Container runtime

El container runtime es el software que realmente ejecuta los contenedores. El kubelet no crea contenedores directamente; se comunica con el runtime a traves de la CRI (Container Runtime Interface), un protocolo definido por Kubernetes que estandariza las operaciones de creacion, inicio, detencion y eliminacion de contenedores.

Antes de la CRI, Kubernetes tenia integracion directa con Docker mediante un modulo llamado dockershim. Docker no implementaba la CRI de forma nativa, por lo que se necesitaba un adaptador. A partir de Kubernetes 1.24, dockershim fue eliminado y Docker dejo de ser un runtime soportado directamente. Ahora Kubernetes solo acepta runtimes que implementen la CRI.

### containerd

containerd es el runtime mas utilizado en Kubernetes. Nacio como un componente interno de Docker y fue donado a la CNCF en 2017. Es un daemon liviano que se encarga de la gestion completa del ciclo de vida del contenedor: descarga de imagenes, gestion del sistema de archivos, ejecucion de procesos y aislamiento mediante namespaces y cgroups de Linux.

Al ser el runtime por defecto de Docker y de Kubernetes, containerd se beneficia de la mayor base de instalaciones y del ecosistema mas amplio de herramientas y monitorizacion. Su rendimiento es superior al de Docker porque elimina la capa adicional del demonio de Docker y se comunica directamente con runc (el low-level runtime que realmente crea los contenedores usando syscalls del kernel).

```bash
# Verificar que containerd esta activo
systemctl status containerd

# containerd expone un socket Unix en /run/containerd/containerd.sock
# Las herramientas como ctr y nerdctl permiten interactuar con el
ctr images ls
```

### CRI-O

CRI-O es un runtime disenado especificamente para Kubernetes, desarrollado originalmente por Red Hat. Su objetivo es ser una implementacion minima de la CRI sin capas de abstraccion innecesarias. A diferencia de containerd, que puede usarse independientemente de Kubernetes, CRI-O solo funciona como runtime para Kubernetes.

CRI-O utiliza runc o crun como low-level runtime y soporta la descarga de imagenes desde registros OCI-compatibles. Su superficie de ataque es menor que la de containerd al tener menos funcionalidades, lo que lo hace atractivo en entornos donde la seguridad y la minimalidad son prioritarias.

```bash
# Verificar CRI-O
systemctl status crio

# El socket de CRI-O esta en /var/run/crio/crio.sock
crictl pods
crictl images
```

### Otros runtimes

gVisor (Google) proporciona una capa de aislamiento adicional ejecutando el contenedor en un kernel de espacio de usuario, lo que anade una barrera de seguridad entre el contenedor y el kernel del host. Kata Containers ejecuta cada contenedor dentro de una maquina virtual liviana, combinando la velocidad de los contenedores con el aislamiento de las VMs. Ambos implementan la CRI y pueden usarse como runtime en Kubernetes, generalmente configurados como runtime clase para cargas de trabajo especificas que requieren mayor aislamiento.

```yaml
# RuntimeClass para usar gVisor en pods especificos
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-pod
spec:
  runtimeClassName: gvisor
  containers:
    - name: app
      image: mi-app
```

> La eliminacion de dockershim no cambio la experiencia del administrador: los comandos de Docker siguen funcionando en los nodos, pero Kubernetes ya no usa el demonio de Docker. Las herramientas `crictl` (para runtimes CRI) y `nerdctl` (compatible con Docker CLI pero sobre containerd) son los reemplazos naturales para depurar contenedores en el nodo.

## Kube-proxy

kube-proxy es el componente responsable de implementar el modelo de red de Kubernetes en cada nodo. Su funcion es traducir las reglas de los Services de Kubernetes en regulas de red reales que dirijan el trafico hacia los pods correctos.

kube-proxy se ejecuta como un DaemonSet en cada nodo y observa constantemente el API server en busca de cambios en Services y Endpoints. Cuando se crea o modifica un Service, kube-proxy actualiza las reglas de red del nodo para que el trafico dirigido a la IP del Service (ClusterIP) se redirija a las IPs de los pods backend.

kube-proxy soporta tres modos de operacion. El modo `userspace` (obsoleto) procesaba el trafico en espacio de usuario, redirigiendolo mediante un proxy a nivel de aplicacion. El modo `iptables` (predeterminado hasta Kubernetes 1.28) crea reglas de iptables en el nodo para realizar NAT (Network Address Translation) del destino: la IP del Service se traduce a la IP de un pod seleccionado aleatoriamente. El modo `ipvs` (recomendado para clusters grandes) utiliza IP Virtual Server del kernel de Linux, que ofrece mejor rendimiento, algoritmos de balanceo mas sofisticados (round-robin, least connection, hash) y menor latencia que iptables.

```bash
# Ver el modo de kube-proxy
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# En modo iptables, se pueden ver las reglas creadas
iptables -t nat -L KUBE-SERVICES

# En modo ipvs, las reglas se ven con ipvsadm
ipvsadm -Ln
```

A partir de Kubernetes 1.26, se introdujo el modo `nftables` como alternativa moderna a iptables. nftables es el sustituto de iptables en el kernel de Linux y ofrece una sintaxis mas limpia y mejor rendimiento. Sin embargo, a dia de hoy iptables e ipvs siguen siendo los modos mas utilizados en produccion.

> kube-proxy no es un proxy en el sentido tradicional (no recibe conexiones y las reenvia). Es un gestor de reglas de red. El trafico fluye directamente entre el cliente y el pod a traves de las reglas de iptables o ipvs, sin pasar por un proceso intermedio. Esto elimina un salto de red y mejora la latencia.

```bash
# Flujo del trafico con kube-proxy en modo iptables
# 1. Cliente envia paquete a 10.96.0.1:80 (ClusterIP del Service)
# 2. Iptables intercepta el paquete (regla KUBE-SERVICES)
# 3. Iptables hace DNAT a 10.244.1.5:8080 (IP real de un pod)
# 4. El paquete llega directamente al pod
```

