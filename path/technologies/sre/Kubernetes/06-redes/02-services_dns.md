# Services y DNS en Kubernetes

Un cluster de Kubernetes sin servicios de descubrimiento de nombres seria inoperable a cualquier escala que supere unos pocos pods. El modelo de red plano garantiza que cualquier pod pueda alcanzar a cualquier otro por IP, pero las IPs son efimeras: un pod muere, otro nace en su lugar con una IP distinta, y las referencias hardcodeadas quedan rotas. Los Services y el sistema DNS interno resuelven este problema de raiz, proporcionando nombres estables y balanceo de carga para conjuntos de pods que cambian constantemente.

El mecanismo combina dos capas que actuan al unisono. Por un lado, el recurso Service define un nombre logico y un conjunto de pods objetivo mediante selectores de etiquetas. Por otro lado, CoreDNS registra automaticamente cada Service como una entrada DNS y la propaga a todo el cluster. El resultado es que cualquier pod puede referirse a otro por un nombre sencillo como `api` o `base-datos`, sin necesidad de conocer direcciones IP concretas ni de gestionar manualmente tablas de servicios.

## El recurso Service

Un Service es una abstraccion que identifica un conjunto de pods (tipicamente gestionados por un Deployment) y define una politica de acceso a ellos. Cuando se crea un Service, Kubernetes le asigna una IP virtual (ClusterIP) que permanece estable durante toda la vida del Service, independientemente de cuantos pods se creen o destruyan.

El Service utiliza un selector de etiquetas para determinar que pods forman parte de el. Los pods que coinciden con el selector se registran automaticamente como endpoints del Service. Cuando un pod se elimina, su endpoint se retira; cuando un nuevo pod coincide con el selector, se agrega como endpoint. Este proceso es continuo y no requiere intervencion manual.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
spec:
  selector:
    app: api
    version: v2
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
```

Los Services se clasifican en varios tipos segun como se exponen:

- **ClusterIP**: el valor por defecto. Asigna una IP virtual accesible solo dentro del cluster. Util para comunicacion interna entre microservicios.
- **NodePort**: ademas de la ClusterIP, abre un puerto en todos los nodos del cluster. El trafico que llega a `<nodo-ip>:<puerto>` se redirige al Service. Util para desarrollo o para integraciones que requieren acceso externo sin balanceador de carga.
- **LoadBalancer**: provisiona un balanceador de carga en el proveedor cloud (AWS ELB, Azure LB, GCP LB) y enruta el trafico externo al Service. Es la forma canonica de exponer servicios al exterior en entornos cloud.
- **ExternalName**: no crea IP virtual ni balanceo. Devuelve un registro CNAME que apunta a un nombre DNS externo. Util para integrar servicios externos (como una base de datos SaaS) como si fueran recursos del cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

## Balanceo de carga interno

Cuando un pod envía una peticion a la ClusterIP de un Service, el trafico no llega directamente a la IP virtual (que no existe como interfaz fisica). En su lugar, kube-proxy intercepta el trafico en el nodo emisor y lo redirige a uno de los pods que cumplen el selector del Service, aplicando balanceo de carga por defecto round-robin.

Kube-proxy configura reglas en iptables (o IPVS) para cada ClusterIP. Cada regla redirige los paquetes destinados a la IP virtual hacia las IPs reales de los pods, con una probabilidad equitativa para cada destino. Este balanceo ocurre a nivel de conexion (no de peticion), lo que significa que todas las peticiones de una misma conexion TCP van al mismo pod.

```bash
# Ver la IP virtual asignada a un Service
kubectl get svc api-svc

# NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# api-svc   ClusterIP   10.96.12.5     <none>        80/TCP    5m

# Ver los endpoints reales detras del Service
kubectl get endpoints api-svc

# NAME      ENDPOINTS                               AGE
# api-svc   10.244.1.5:8080,10.244.2.8:8080,10.244.3.2:8080   5m
```

## Resolucion DNS con CoreDNS

Kubernetes incluye un sistema de DNS interno basado en CoreDNS, un servidor DNS ligero y extensible escrito en Go. CoreDNS se despliega como un Deployment en el namespace `kube-system` y se expone a traves de un Service llamado `kube-dns` en la direccion `10.96.0.10` (aunque la IP puede variar segun la configuracion del cluster).

Cuando un pod se crea, Kubernetes configura su resolucion DNS para que apunte a CoreDNS. Esto se logra inyectando el archivo `/etc/resolv.conf` dentro del contenedor con las siguientes lineas:

```
nameserver 10.96.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
```

Gracias a esto, cualquier pod puede resolver nombres de servicios usando unicamente su nombre corto. Un pod en el namespace `produccion` puede acceder al servicio `api-svc` simplemente como `api-svc`, y CoreDNS lo resuelve automaticamente como `api-svc.produccion.svc.cluster.local`.

```bash
# Desde un pod, resolver el nombre de un servicio
kubectl exec -it pod-ejemplo -- nslookup api-svc

# Server:    10.96.0.10
# Address:   10.96.0.10#53
# Name:      api-svc.default.svc.cluster.local
# Address:   10.96.12.5
```

El formato completo de los nombres DNS de los Services sigue el patron:

```
<nombre-service>.<namespace>.svc.<cluster-domain>
```

Donde `cluster-domain` es `cluster.local` por defecto. Esto permite que servicios de diferentes namespaces se referencien mutuamente usando nombres completos: `base-datos.produccion.svc.cluster.local`.

## Service discovery automatica

El descubrimiento de servicios en Kubernetes es automatico porque CoreDNS utiliza el API server como fuente de datos. Cuando se crea un Service, el controlador de Endpoints registra los pods que coinciden con el selector y actualiza el objeto EndpointSlice. CoreDNS, que observa los cambios en estos recursos a traves del API server, anade o elimina los registros DNS correspondientes en tiempo real.

Esto significa que no hay un paso manual de registro. El flujo completo es:

1. Se crea un Deployment con tres replicas de `api`.
2. Se etiquetan los pods con `app: api`.
3. Se crea un Service con `selector.app: api`.
4. Kubernetes crea automaticamente los Endpoints apuntando a las IPs de los tres pods.
5. CoreDNS detecta el nuevo Service y anade un registro A que apunta a la ClusterIP.
6. Cualquier pod del cluster puede resolver `api-svc` inmediatamente.

```bash
# Ver los EndpointSlice creados automaticamente
kubectl get endpointslices

# NAME                  ADDRESSTYPE   PORTS     ENDPOINTS                    AGE
# api-svc-xxxxx         IPv4          8080      10.244.1.5,10.244.2.8,10.244.3.2  5m
```

## Headless Services

Un headless Service es un Service que no tiene ClusterIP. Se crea especificando `clusterIP: None` en su especificacion. En lugar de una IP virtual con balanceo de carga, un headless Service hace que CoreDNS devuelva directamente las IPs de todos los pods que coinciden con el selector, permitiendo que el cliente decida como balancear o seleccionar el destino.

Los headless services son esenciales para aplicaciones que necesitan conocer la identidad individual de cada pod, como bases de datos distribuidas (Cassandra, Elasticsearch, MongoDB), sistemas de mensajeria (Kafka, RabbitMQ) o cualquier workload stateful donde cada replica tiene un rol distinto (primario, secundario, particion).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
spec:
  clusterIP: None
  selector:
    app: cassandra
  ports:
    - port: 9042
      targetPort: 9042
```

Con un headless service, la resolucion DNS devuelve todas las IPs de los pods en lugar de una unica IP virtual:

```bash
# Resolver un headless service
kubectl exec -it pod-ejemplo -- nslookup cassandra

# Server:    10.96.0.10
# Address:   10.96.0.10#53
# Name:      cassandra.default.svc.cluster.local
# Address:   10.244.1.10
# Address:   10.244.2.11
# Address:   10.244.3.12
```

Cuando se combina un headless service con un StatefulSet, los pods reciben ademas nombres DNS individuales con el formato `<nombre-pod>.<nombre-service>.<namespace>.svc.cluster.local`. Esto permite que cada pod tenga una identidad de red estable y predecible, incluso si se recrea en un nodo distinto.

```bash
# En un StatefulSet con headless service, cada pod tiene su propio nombre DNS
kubectl exec -it cassandra-0 -- nslookup cassandra-1.cassandra.default.svc.cluster.local
```

> Los headless services no proporcionan balanceo de carga. Si el cliente no implementa su propia logica de seleccion (como round-robin DNS o un driver de base de datos que gestione las conexiones), el trafico podria concentrarse en un solo pod.

## Consideraciones practicas sobre DNS

CoreDNS se puede configurar mediante un ConfigMap llamado `coredns` en `kube-system`. Las configuraciones habituales incluyen la integracion con DNS externos (forwarding a otros servidores), el cacheo de respuestas, y el uso de plugins personalizados.

```bash
# Ver la configuracion actual de CoreDNS
kubectl get configmap coredns -n kube-system -o yaml

# apiVersion: v1
# kind: ConfigMap
# metadata:
#   name: coredns
#   namespace: kube-system
# data:
#   Corefile: |
#     .:53 {
#         errors
#         health {
#            lameduck 5s
#         }
#         ready
#         kubernetes cluster.local in-addr.arpa ip6.arpa {
#            pods insecure
#            fallthrough in-addr.arpa ip6.arpa
#            ttl 30
#         }
#         prometheus :9153
#         forward . /etc/resolv.conf
#         cache 30
#         loop
#         reload
#         loadbalance
#     }
```

La resolucion DNS puede fallar si los pods del CoreDNS no estan operativos. Es recomendable ejecutar al menos dos replicas de CoreDNS y distribuir las cargas de trabajo de forma que el DNS nunca sea un punto unico de fallo. Los problemas de DNS en Kubernetes suelen manifestarse como errores de conexion intermitentes entre servicios, que pueden diagnosticarse con herramientas como `nslookup`, `dig` o `dnsutils` desde un pod de depuracion.

