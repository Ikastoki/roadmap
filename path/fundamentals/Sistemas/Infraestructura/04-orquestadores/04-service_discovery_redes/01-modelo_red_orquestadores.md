# Modelo de Red en Orquestadores

Los orquestadores de contenedores transforman radicalmente la forma en que se gestionan las comunicaciones de red al introducir un modelo donde cada unidad de trabajo —un pod, un task— recibe una dirección IP única dentro de la red del clúster. Este enfoque elimina la dependencia de tablas NAT y el mapeo de puertos del host, permitiendo que las comunicaciones entre unidades de trabajo se realizen de forma directa, como si cada contenedor residiera en su propio host de red. La consecuencia más inmediata es que la topología de red deja de depender de la infraestructura física y pasa a ser una abstracción completamente gestionada por el orquestador.

En Kubernetes, cada pod se comporta como una interfaz de red independiente dentro de la red plana del clúster. Todos los contenedores que comparten un mismo pod comparten también el mismo namespace de red, lo que significa que pueden comunicarse entre sí mediante `localhost` y comparten los mismos puertos. Esta diseño resuelve el problema de la comunicación entre contenedores acoplados sin necesidad de redes auxiliares o sidecar de red dedicados.

## Red plana y direccionamiento IP

El orquestador construye una red plana (flat network) donde todos los pods reciben direcciones IP routables directamente entre sí. No existe segmentación por host físico ni por nodo; un pod en el nodo A puede comunicarse con un pod en el nodo B usando únicamente su IP, sin intermediarios de NAT. Este modelo depende de un plugin CNI (Container Network Interface) que implementa la capa de red subyacente.

```yaml
# En Kubernetes, cada pod obtiene una IP del CIDR asignado al clúster
# Esta IP es accesible desde cualquier otro pod del clúster
apiVersion: v1
kind: Pod
metadata:
  name: mi-app
  labels:
    app: mi-app
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
```

> La IP del pod es asignada por el CNI plugin al momento de la creación del pod. No hay DHCP involucrado; la IP se reserva en la interfaz `eth0` del pod dentro del namespace de red antes de que cualquier contenedor inicie.

## Namespace de red compartido

Dentro de un mismo pod, todos los contenedores comparten la misma interfaz de red. Esto significa que un contenedor escuchando en el puerto 8080 es accesible desde otro contenedor del mismo pod como `localhost:8080`. Esta compartición facilita patrones arquitectónicos como sidecars, donde un contenedor secundario gestiona tareas auxiliares (logging, proxy inverso, certificates) sin necesidad de red externa.

```bash
# Ver la configuracion de red de un pod
kubectl exec -it mi-app -- ip addr show eth0

# Ver los puertos que estan escuchando dentro del pod
kubectl exec -it mi-app -- ss -tlnp

# Comunicarse entre contenedores del mismo pod via localhost
kubectl exec -it mi-app -c sidecar -- curl http://localhost:8080
```

El kernel de Linux crea un namespace de red al inicio del pod. Cada contenedor dentro del pod se une a ese namespace mediante una interfaz virtual `veth pair`: un extremo reside en el namespace del pod y el otro en el bridge o red del nodo. Cuando el pod se destruye, el namespace y todas sus interfaces se liberan.

## Comunicacion entre pods en distintos nodos

Cuando dos pods residen en nodos distintos, la comunicacion se realiza a traves de la red overlay o de la red subyacente del clúster. El plugin CNI se encarga de enrutar los paquetes entre nodos sin que el usuario tenga que gestionar rutas manualmente. El orquestador garantiza que cada pod pueda alcanzar a cualquier otro pod independientemente de su ubicacion fisica.

```bash
# Ver la tabla de rutas del clúster desde un nodo
kubectl exec -it mi-app -- ip route show

# Resultado tipico de un clúster con Calico:
# default via 169.254.1.1 dev eth0
# 10.244.0.0/24 via 10.244.0.1 dev eth0
# 10.244.1.0/24 via 10.244.0.1 dev eth0  <-- pod en otro nodo
```

```yaml
# Verificar la red del clúster con un diagnostic tool
apiVersion: v1
kind: Pod
metadata:
  name: netdiag
  namespace: default
spec:
  containers:
    - name: netdiag
      image: nicolaka/netshoot
      command: ["/bin/bash"]
      args: ["-c", "sleep infinity"]
```

> Los plugins CNI como Calico, Flannel o Cilium implementan distintas estrategias de red (VXLAN, BGP, eBPF). La eleccion del plugin determina como se enrutan los paquetes entre nodos y que funcionalidades de red estan disponibles (network policies, cifrado, observabilidad).

## Comunicacion con servicios externos

Para que un pod acceda a servicios fuera del clúster, el orquestador proporciona mecanismos de salida (egress) que traducen las direcciones internas a la red externa. En Kubernetes, el kube-proxy gestiona las reglas NAT en cada nodo para que el trafico saliente del pod se enrute correctamente hacia la red exterior. La politica de red puede restringir o permitir estas comunicaciones de salida.

```bash
# Ver las reglas de NAT del kube-proxy en un nodo
sudo iptables -t nat -L KUBE-SERVICES -n

# Verificar conectividad externa desde un pod
kubectl exec -it mi-app -- curl -s https://api.ejemplo.com/v1/health

# Verificar reglas de egress configuradas
kubectl get networkpolicy --all-namespaces -o wide
```

La coexistencia de comunicacion interna (pod a pod) y externa (pod a Internet) dentro del mismo modelo de red plana es una de las fortalezas del orquestador. El administrador puede definir con precision que trafico es interno y que trafico puede salir del clúster, manteniendo el principio de minimo privilegio incluso a nivel de red.

