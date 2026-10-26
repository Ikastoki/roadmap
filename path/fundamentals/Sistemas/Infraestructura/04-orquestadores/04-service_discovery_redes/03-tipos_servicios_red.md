# Tipos de Servicios de Red

Un servicio en el contexto de orquestadores es una abstraccion que define un conjunto logico de pods y una politica de acceso a ellos. Los servicios resuelven el problema de la efimeridad de los pods: mientras un pod tiene una IP temporal que cambia al reiniciarse, un servicio mantiene una IP estable y un nombre DNS consistente que enruta el trafico hacia los endpoints saludables. Sin embargo, no todas las necesidades de comunicacion son iguales, por lo que Kubernetes define varios tipos de servicios con distintas capacidades de exposicion.

La eleccion del tipo de servicio determina como el trafico fluye dentro y fuera del clúster. Un servicio interno que solo necesita comunicar componentes entre si tiene requisitos completamente diferentes a uno que debe aceptar conexiones desde Internet. Cada tipo ofrece un equilibrio distinto entre visibilidad, control y complejidad operativa.

## ClusterIP: servicio interno

El tipo `ClusterIP` es el por defecto en Kubernetes. Asigna al servicio una IP virtual unica dentro del rango de direcciones del clúster, accesible unicamente desde dentro del clúster. Este tipo es ideal para servicios backend, bases de datos o componentes internos que no deben ser expuestos al exterior.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-backend
  namespace: produccion
spec:
  type: ClusterIP
  selector:
    app: api-backend
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

```bash
# Verificar el servicio y su IP
kubectl get svc api-backend -n produccion

# NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
# api-backend   ClusterIP   10.96.45.12    <none>        80/TCP

# Probar conectividad desde un pod
kubectl exec -it mi-app -- curl http://api-backend.produccion.svc.cluster.local
```

> kube-proxy implementa la IP virtual del ClusterIP mediante reglas iptables o IPVS en cada nodo. Cuando un pod envia trafico a la IP del servicio, las reglas redirigen el paquete a uno de los pods backends seleccionados por el label `selector`.

## NodePort: exposicion por nodo

El tipo `NodePort` expone un puerto fijo en todos los nodos del clúster. Cualquier trafico que llegue a `<IP-del-nodo>:<NodePort>` se redirige internamente hacia los pods seleccionados por el servicio. Este tipo permite el acceso externo sin un balanceador de carga dedicado, util para entornos de desarrollo o pruebas.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: produccion
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 3000
      nodePort: 30080   # Rango valido: 30000-32767
      protocol: TCP
```

```bash
# Ver el puerto NodePort asignado
kubectl get svc frontend -n produccion

# NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# frontend   NodePort   10.96.78.34     <none>        80:30080/TCP   5m

# Acceder al servicio desde fuera del clúster
curl http://192.168.1.10:30080

# Ver los nodos disponibles
kubectl get nodes -o wide
```

> El puerto NodePort debe estar abierto en el firewall del clúster (grupos de seguridad en la nube). Si el trafico ingresa por un nodo que no tiene pods del servicio, kube-proxy lo enruta hacia un nodo que si los tiene. Esto puede introducir un salto de red adicional.

## LoadBalancer: integracion con proveedores de nube

El tipo `LoadBalancer` crea automaticamente un balanceador de carga externo en el proveedor de nube (AWS ELB, GCP Load Balancer, Azure Load Balancer). El balanceador recibe una IP externa y distribuye el trafico entre los nodos del clúster, que a su vez redirigen hacia los pods correspondientes. Es el tipo recomendado para servicios de produccion expuestos al exterior.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-public
  namespace: produccion
  annotations:
    # Anotaciones especificas del proveedor
    # AWS: service.beta.kubernetes.io/aws-load-balancer-type: nlb
    # GCP: cloud.google.com/load-balancer-type: Internal
spec:
  type: LoadBalancer
  selector:
    app: api-public
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

```bash
# Ver la IP externa asignada al balanceador
kubectl get svc api-public -n produccion

# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
# api-public   LoadBalancer   10.96.12.45    34.102.200.15    443:31234/TCP,80:30123/TCP

# Ver las anotaciones del balanceador creado
kubectl get svc api-public -n produccion -o jsonpath='{.metadata.annotations}'
```

> El tipo LoadBalancer tiene un costo asociado: cada servicio de este tipo crea un balanceador de carga externo. En un clúster con muchos servicios expuestos, esto puede generar costos significativos. El uso de un Ingress (ver siguiente tema) permite compartir un unico balanceador para multiples servicios.

## ExternalName: mapeo a servicio externo

El tipo `ExternalName` crea un alias DNS (registro CNAME) hacia un servicio que reside fuera del clúster. No hay seleccion de pods ni endpoints gestionados por Kubernetes. Este tipo es util para integrar servicios externos (APIs de terceros, bases de datos gestionadas) en el mismo namespace de servicios internos, manteniendo una interfaz uniforme.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-externa
  namespace: produccion
spec:
  type: ExternalName
  externalName: api.tercero.com
```

```bash
# Verificar el registro CNAME
kubectl exec -it mi-app -- nslookup api-externa.produccion.svc.cluster.local

# Resultado:
# api-externa.produccion.svc.cluster.local  canonical name = api.tercero.com
# Name:      api.tercero.com
# Address:   203.0.113.50
```

> Un servicio ExternalName no puede tener un selector ni exponer puertos. Si se necesita combinar un servicio externo con puertos especificos, se debe crear un servicio de tipo ClusterIP con un `Endpoints` apuntando manualmente a la IP externa.

## Headless: acceso directo a instancias

Un servicio headless se configura estableciendo `clusterIP: None`. En lugar de asignar una IP virtual, una consulta DNS a un servicio headless retorna las IPs individuales de todos los pods seleccionados. Esto permite que el cliente implemente su propia logica de balanceo o que acceda a una instancia especifica, como es el caso de bases de datos con replicas (StatefulSets).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-cluster
  namespace: produccion
spec:
  clusterIP: None   # Servicio headless
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

```bash
# Verificar que el servicio no tiene clusterIP
kubectl get svc postgres-cluster -n produccion

# NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# postgres-cluster  ClusterIP   None         <none>        5432/TCP

# Resolver el DNS del servicio headless: retorna IPs individuales
kubectl exec -it mi-app -- nslookup postgres-cluster.produccion.svc.cluster.local

# Resultado:
# Address: 10.244.0.15
# Address: 10.244.1.23
# Address: 10.244.2.8

# Acceder a una instancia especifica via DNS StatefulSet
kubectl exec -it mi-app -- nslookup postgres-0.postgres-cluster.produccion.svc.cluster.local
```

> Los servicios headless son esenciales para StatefulSets, donde cada pod tiene un nombre estable (postgres-0, postgres-1, postgres-2). El cliente puede conectarse a una replica especifica usando su DNS predecible, lo que es critico para bases de datos que necesitan direccionamiento directo a lider y replicas.

