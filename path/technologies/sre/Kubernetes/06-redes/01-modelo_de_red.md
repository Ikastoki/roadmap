# Modelo de red en Kubernetes

Kubernetes no implementa un sistema de red propio, sino que define un modelo abstracto y exigente que cualquier implementacion de red debe satisfacer. Este modelo consta de tres proposiciones fundamentales: cada pod recibe una direccion IP unica en todo el cluster, todos los pods pueden comunicarse entre si sin necesidad de traduccion de direcciones (NAT), y los contenedores dentro de un mismo pod comparten el mismo espacio de red. Estas reglas pueden parecer simples, pero sus implicaciones son profundas: eliminan la necesidad de mapear puertos entre contenedores, hacen que la topologia de red del cluster sea plana desde la perspectiva de los pods, y permiten tratar a cada pod como un nodo de red tradicional dentro de un segmento unico.

El modelo flat network no surge de un capricho arquitectonico. En los sistemas de orquestacion anteriores a Kubernetes, la comunicacion entre contenedores requeria configurar puentes, reglas NAT, o servicios de descubrimiento externos. Cada plataforma inventaba su propio esquema, lo que hacia impracticable la portabilidad de aplicaciones entre entornos. Al estandarizar el modelo de red como parte de la propia plataforma, Kubernetes garantiza que cualquier aplicacion disenada para el cluster pueda comunicarse sin cambios, independientemente del proveedor de red subyacente.

## Cada pod tiene una IP unica

A cada pod se le asigna una direccion IP real y enrutable dentro del cluster, no una direccion virtual detras de un proxy. Esta IP es unica en todo el cluster, lo que significa que no hay colisiones entre pods de diferentes nodos. Desde el punto de vista de la red, cada pod se comporta como una maquina independiente conectada a un mismo segmento Ethernet.

La asignacion de IPs se delega al plugin de red que implementa la CNI (Container Network Interface). El plugin reserva un rango de direcciones (tipicamente un CIDR como `10.244.0.0/16`) y asigna una IP a cada pod cuando se crea. Cuando el pod se destruye, la IP se libera y puede ser reutilizada por otro pod. Este comportamiento descarta cualquier suposicion sobre la permanencia de las direcciones IP, lo que obliga a las aplicaciones a utilizar descubrimiento de servicios en lugar de IPs hardcodeadas.

```bash
# Ver las IPs asignadas a los pods
kubectl get pods -o wide

# NAME                     READY   STATUS    IP           NODE
# frontend-7d4f8b5c-2kx9   1/1     Running   10.244.1.5   nodo-1
# api-6f8d4c8b7c-abc12     1/1     Running   10.244.2.8   nodo-2
```

## Los contenedores del mismo pod comparten red

Los contenedores que forman parte de un mismo pod comparten el mismo espacio de nombres de red (network namespace). Esto significa que comparten la misma direccion IP, el mismo rango de puertos y la misma pila TCP/IP. La comunicacion entre ellos se realiza a traves de `localhost`, sin necesidad de servicios de red ni configuraciones especiales.

Kubernetes implementa esto mediante un contenedor especial llamado `pause`, que se crea antes que los contenedores de aplicacion y mantiene el network namespace del pod. Los contenedores de aplicacion se unen a ese namespace ya existente. Cuando el pod se destruye, el contenedor `pause` mantiene el namespace el tiempo suficiente para que los demas contenedores finalicen correctamente.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-con-sidecar
spec:
  containers:
    - name: app
      image: mi-aplicacion:1.0
      ports:
        - containerPort: 8080
    - name: sidecar
      image: fluentd:latest
      # Ambos contenedores se comunican por localhost:8080
```

> Compartir red no implica compartir sistema de archivos. Si dos contenedores necesitan intercambiar archivos, deben hacerlo a traves de un volumen compartido (emptyDir o similar). La red compartida solo cubre la comunicacion a nivel TCP/UDP.

## Comunicacion entre pods sin NAT

La segunda regla del modelo de red de Kubernetes establece que los pods pueden comunicarse entre si sin traduccion de direcciones. Un pod con IP `10.244.1.5` puede enviar trafico directamente a `10.244.2.8` sin pasar por ningun proxy o regla NAT. Esto es posible porque el plugin de red configura las rutas necesarias en cada nodo para que el trafico entre pods atraviese la infraestructura de red subyacente de forma transparente.

La ausencia de NAT tiene consecuencias importantes. Las aplicaciones ven la direccion IP real del pod emisor, lo que permite logging, auditoria y autenticacion basada en IP de origen sin falsificaciones. Ademas, elimina la sobrecarga de traduccion de direcciones y los problemas asociados a la permanencia de conexiones a traves de NAT.

```bash
# Desde un pod, verificar conectividad directa a otro pod
kubectl exec -it frontend-7d4f8b5c-2kx9 -- ping 10.244.2.8
# 64 bytes from 10.244.2.8: icmp_seq=1 ttl=62 time=0.5 ms
```

## Container Network Interface (CNI)

Kubernetes no implementa el plano de datos de la red. En su lugar, define una interfaz estandar llamada CNI (Container Network Interface) que los proveedores de red implementan. CNI especifica como configurar las interfaces de red de los contenedores, asignar direcciones IP y mantener la conectividad a traves de los nodos del cluster.

Cuando el kubelet crea un pod, invoca al plugin CNI configurado en el nodo. El plugin recibe un identificador unico para el pod y devuelve la configuracion de red resultante: la IP asignada, las rutas y las interfaces virtuales creadas. Este proceso ocurre en milisegundos y no requiere intervencion del administrador.

Existen multiples implementaciones CNI, cada una con enfoques distintos para resolver el mismo problema:

- **Calico**: utiliza BGP (Border Gateway Protocol) para distribuir rutas entre nodos. Ofrece rendimiento cercano al hardware y soporta Network Policies con alta granularidad. Es la opcion mas utilizada en entornos de produccion.
- **Flannel**: crea una overlay network (red superpuesta) usando VXLAN o host-gw. Es simple de configurar pero no soporta Network Policies por si mismo (requiere un componente adicional como Canal).
- **Weave Net**: implementa una malla completa entre nodos con cifrado opcional. Cada nodo conoce la topologia completa y puede enrutar trafico directamente o a traves de otros nodos.
- **Cilium**: utiliza eBPF (extended Berkeley Packet Filter) para implementar red, seguridad y observabilidad en el kernel de Linux. Ofrece rendimiento superior y capacidades avanzadas como servicios mesh y Hubble para monitoreo.

```bash
# Ver el plugin CNI configurado en el cluster
# Los pods del plugin suelen estar en kube-system
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'

# Ejemplo con Calico
# calico-node-xxxxx            1/1     Running   0  24h
# calico-kube-controllers-xxx  1/1     Running   0  24h
```

## Como se implementa la comunicacion nodo a nodo

Cuando un pod en el nodo A necesita comunicarse con un pod en el nodo B, el trafico debe salir del nodo A, viajar por la red fisica (o virtual) y llegar al nodo B. El plugin CNI se encarga de que esto ocurra de forma transparente para los contenedores.

En una overlay network (Flannel con VXLAN), los paquetes del pod se encapsulan dentro de paquetes UDP y se envian al nodo destino, donde se desencapsulan y entregan al pod receptor. Este encapsulamiento anade overhead de cabeceras pero funciona en cualquier infraestructura de red sin necesidad de configurar los switches fisicos.

En una implementacion basada en rutas (Calico con BGP), cada nodo anuncia las subredes de sus pods a traves del protocolo BGP. Los routers fisicos o virtuales aprenden estas rutas y el trafico viaja directamente de un nodo a otro sin encapsulamiento. Esto ofrece mejor rendimiento pero requiere que la infraestructura de red soporte BGP.

```bash
# Ver las rutas establecidas por el plugin CNI en un nodo
ip route | grep 10.244

# 10.244.1.0/24 dev cali-xxxxx scope link
# 10.244.2.0/24 via 192.168.1.2 dev eth0
# 10.244.3.0/24 via 192.168.1.3 dev eth0
```

## kube-proxy y la magic del Service

El modelo de red plano resuelve la comunicacion entre pods, pero no cubre un problema fundamental: los pods son efimeros y sus IPs cambian constantemente. Si una aplicacion necesita comunicarse con un conjunto de pods que van y vienen, necesita un nombre estable que no dependa de una IP concreta.

Aqui entra kube-proxy, el componente de red que se ejecuta en cada nodo. kube-proxy observa los cambios en los recursos Service y EndpointSlice a traves del API server, y actualiza las reglas de red en el nodo para que las peticiones dirigidas a la IP virtual del Service se distribuyan entre los pods que lo componen. kube-proxy soporta tres modos de operacion: userspace (obsoleto), iptables (el mas comun) e IPVS (para clusters con muchos servicios).

```bash
# Ver las reglas iptables que kube-proxy configura
# (especialmente la cadena KUBE-SERVICES)
sudo iptables -t nat -L KUBE-SERVICES | head -20

# Ver los endpoints que respaldan un servicio
kubectl get endpoints mi-servicio
```

> El modelo de red de Kubernetes no es opcional ni negociable. Cualquier implementacion que pretenda ser compatible debe cumplir las tres reglas: IP unica por pod, comunicacion sin NAT y namespace de red compartido entre contenedores del mismo pod. Los plugins CNI que no respeten estas reglas simplemente no funcionaran correctamente.

