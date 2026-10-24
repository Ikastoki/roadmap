# Services

Los pods son efimeros. Cuando un pod muere y el ReplicaSet lo reemplaza, el nuevo pod recibe una direccion IP diferente. Si otros componentes del sistema necesitan comunicarse con el, no pueden depender de una IP fija. Un Service resuelve este problema proporcionando una abstraccion de red estable: una IP virtual y un nombre DNS que permanecen constantes independientemente de los pods subyacentes.

Un Service es un recurso de Kubernetes que define un conjunto logico de pods (mediante un selector de labels) y una politica para acceder a ellos. Kubernetes asigna al Service una IP del rango de servicios del cluster y, si CoreDNS esta configurado, un nombre DNS automatico. El trafico dirigido a esa IP o nombre se distribuye entre los pods que coinciden con el selector.

## ClusterIP

ClusterIP es el tipo de Service por defecto. Expone el servicio unicamente dentro del cluster, utilizando una IP virtual interna. Es el tipo adecuado para comunicacion entre servicios internos: un backend que llama a la API, un worker que consume mensajes de una cola, un frontend que se comunica con el backend.

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
      protocol: TCP
  type: ClusterIP
```

```bash
# Crear el servicio
kubectl apply -f service-clusterip.yaml

# Ver el servicio y su IP asignada
kubectl get services

# Desde cualquier pod del cluster, se accede por nombre o IP
kubectl run curl --image=curlimages/curl -it --rm -- sh
# curl http://api-service:80/health
# curl http://10.96.0.10:80/health
```

El campo `port` es el puerto en el que se expone el servicio. `targetPort` es el puerto del contenedor al que se redirige el trafico. Si no se especifica `targetPort`, se asume el mismo valor que `port`. Si los pods exponen multiples puertos, pueden definirse varios entries en `ports`, cada uno con un `name` unico.

## NodePort

NodePort extiende ClusterIP y ademas expone el servicio en un puerto estatico en cada nodo del cluster. El rango de puertos NodePort por defecto es 30000-32767. El trafico que llega a `http://<nodo-ip>:<node-port>` se redirige al servicio correspondiente.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-nodeport
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

Si no se especifica `nodePort`, Kubernetes asigna uno aleatorio dentro del rango configurado. NodePort es util para desarrollo local (acceso a servicios sin balanceador de carga) o cuando se necesita exponer un servicio detras de un balanceador externo que no es gestionado por Kubernetes.

```bash
# Obtener la IP de un nodo
kubectl get nodes -o wide

# Acceder al servicio desde fuera del cluster
curl http://192.168.1.100:30080/health
```

## LoadBalancer

LoadBalancer es el tipo que integra Kubernetes con el balanceador de carga del proveedor cloud (AWS ELB, GCP GLB, Azure Load Balancer). Cuando se crea un Service de tipo LoadBalancer, Kubernetes provisiona automaticamente un balanceador de carga externo y configura las reglas necesarias para reenviar trafico a los nodos del cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-lb
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

```bash
# Crear el servicio
kubectl apply -f service-loadbalancer.yaml

# Esperar a que el balanceador se provisione
kubectl get svc api-lb --watch
# NAME      TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# api-lb    LoadBalancer   10.96.0.11     pending       80:32001/TCP   5s
# api-lb    LoadBalancer   10.96.0.11     a1b2c3d4...   80:32001/TCP   30s
```

El campo `EXTERNAL-IP` contiene la direccion publica del balanceador de carga una vez provisionado. Los proveedores cloud facturan por cada balanceador activo, por lo que no conviene crear LoadBalancers innecesarios.

En entornos locales (Minikube, Kind), LoadBalancer no puede provisionar un balanceador real, pero se puede acceder al servicio mediante `kubectl port-forward` o usando el comando `minikube tunnel`.

## ExternalName

ExternalName es el unico tipo de Service que no tiene selector ni direccion IP propia. En lugar de enrutar a pods, devuelve un registro CNAME en la resolucion DNS del cluster. Se utiliza para exponer servicios externos (bases de datos, APIs de terceros) como si fueran servicios internos del cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mi-base-de-datos.ejemplo.com
```

```bash
# Desde cualquier pod del cluster
kubectl run curl --image=curlimages/curl -it --rm -- sh
# curl http://external-db.default.svc.cluster.local:5432
# Se resuelve como CNAME a mi-base-de-datos.ejemplo.com
```

ExternalName no realiza proxy ni balanceo. Es puramente una resolucion DNS. Si el servicio externo cambia de IP, solo hay que actualizar el registro DNS externo, no el manifiesto de Kubernetes.

## Selector y endpoints

El Service utiliza el campo `selector` para determinar que pods reciben trafico. Los pods cuyas labels coinciden con el selector se incluyen automaticamente en el conjunto de endpoints del servicio.

```yaml
spec:
  selector:
    app: api
    environment: production
```

Kubernetes crea un recurso Endpoints (o EndpointSlice en versiones recientes) que mantiene la lista actualizada de IPs y puertos de los pods que cumplen el selector. Cuando un pod se crea o destruye, el controlador de endpoints actualiza este recurso en tiempo real.

```bash
# Ver los endpoints del servicio
kubectl get endpoints api-service
# NAME          ENDPOINTS                            AGE
# api-service   10.244.1.5:8080,10.244.2.7:8080      5m
```

Si el selector del servicio no coincide con ningun pod, el servicio existe pero no tiene endpoints. Las peticiones a su IP o DNS se rechazan (connection refused). Un servicio sin selector puede tener endpoints gestionados manualmente, lo que permite enrutar trafico a direcciones externas sin usar ExternalName.

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db
subsets:
  - addresses:
      - ip: 203.0.113.10
    ports:
      - port: 5432
```

## DNS interno

Kubernetes incorpora CoreDNS como servicio de descubrimiento de nombres dentro del cluster. Cada Service recibe automaticamente un registro DNS con el formato:

```
<nombre-servicio>.<namespace>.svc.cluster.local
```

```bash
# Resolver un servicio desde cualquier pod
kubectl exec -ti any-pod -- nslookup api-service
# Name:   api-service.default.svc.cluster.local
# Address: 10.96.0.10

# O usando herramientas de red
kubectl exec -ti any-pod -- curl http://api-service:80
```

El nombre corto (sin el sufijo `.svc.cluster.local`) funciona dentro del mismo namespace. Para acceder a un servicio de otro namespace, debe usarse el nombre completo o el patron `<servicio>.<namespace>`:

```
# Mismo namespace
curl http://api-service:80

# Distinto namespace
curl http://api-service.produccion:80
curl http://api-service.produccion.svc.cluster.local:80
```

CoreDNS tambien resuelve pods individuales mediante registros A si se configuran headless services (servicios sin IP cluster, con `clusterIP: None`). Esto es util para StatefulSets donde cada pod necesita una identidad de red unica.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: statefulset-svc
spec:
  clusterIP: None
  selector:
    app: database
```

Con un headless service, cada pod recibe un registro DNS del tipo `<pod-name>.<service-name>.<namespace>.svc.cluster.local`, permitiendo conexiones directas a pods individuales.

## Puertos multiples y naming

Un servicio puede exponer multiples puertos. Cada puerto debe tener un nombre unico cuando hay mas de uno definido.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 9090
      targetPort: 9090
  type: ClusterIP
```

Los nombres de puerto son obligatorios en servicios con multiples puertos y deben coincidir con los nombres definidos en el pod. Esto permite que el protocolo de cada puerto sea autodescriptivo y que las politicas de red puedan referirse a puertos por nombre en lugar de numero.

