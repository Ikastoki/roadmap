# Service Discovery

En un entorno de contenedores orquestados, las instancias de un servicio aparecen y desaparecen dinamicamente. Un pod que falla es reemplazado automaticamente con una nueva IP; un escalado horizontal genera nuevas instancias con direcciones diferentes. El service discovery resuelve el problema fundamental de como un servicio localiza a otro sin depender de direcciones IP hardcodeadas, proporcionando un mecanismo de resolucion de nombres que se adapta automaticamente a los cambios en la composicion del clúster.

El orquestador implementa service discovery principalmente a traves de un sistema DNS interno que asocia nombres logicos a conjuntos de endpoints dinamicos. Cuando un servicio cambia —una instancia nueva aparece, una fallida desaparece— el registro DNS se actualiza automaticamente, garantizando que los clientes siempre resuelvan nombres hacia instancias saludables.

## DNS interno del clúster

Kubernetes incluye un servidor DNS interno (CoreDNS) que se ejecuta como un pod dentro del clúster. Todos los servicios y pods registran automaticamente sus nombres en este servidor DNS, siguiendo una convencion de nomenclatura jerarquica. Un servicio en el namespace `produccion` con nombre `api-backend` es accesible como `api-backend.produccion.svc.cluster.local`.

```bash
# Ver el servicio DNS interno del clúster
kubectl get svc kube-dns -n kube-system

# Verificar la resolucion DNS desde dentro de un pod
kubectl exec -it mi-app -- nslookup api-backend.produccion.svc.cluster.local

# Resultado:
# Name:      api-backend.produccion.svc.cluster.local
# Address:   10.96.45.12

# Verificar la resolucion DNS con dig para mas detalle
kubectl exec -it mi-app -- dig api-backend.produccion.svc.cluster.local +short
# 10.96.45.12
```

> La abreviatura `svc.cluster.local` es el dominio de busqueda por defecto. Dentro de un namespace, un servicio puede referenciarse simplemente por su nombre: `api-backend`. El DNS interno resuelve el nombre completo automaticamente.

El servidor CoreDNS mantiene una zona DNS para el dominio `cluster.local` y genera registros de tipo A (IPv4) y AAAA (IPv6) para cada servicio. Cuando un servicio cambia sus endpoints —por ejemplo, durante un rolling update— CoreDNS actualiza los registros de forma casi instantanea, propagando el cambio a todos los resolvers del clúster.

## Variables de entorno como mecanismo de discovery

Ademas del DNS, Kubernetes inyecta automaticamente variables de entorno en cada pod recien creado con la informacion de todos los servicios visibles en su namespace. Para cada servicio llamado `mi-servicio` en el namespace `default`, se crean las variables `MI_SERVICIO_SERVICE_HOST` y `MI_SERVICIO_SERVICE_PORT` con la IP del servicio y el puerto expuesto.

```yaml
# Pod que consume un servicio a traves de variables de entorno
apiVersion: v1
kind: Pod
metadata:
  name: consumidor
  namespace: default
spec:
  containers:
    - name: app
      image: mi-app:latest
      env:
        # Variable inyectada automaticamente por Kubernetes
        # para el servicio "api-backend" en el mismo namespace
        - name: API_BACKEND_SERVICE_HOST
          valueFrom:
            configMapKeyRef:
              name: mi-config
              key: api-host
```

```bash
# Ver todas las variables de entorno de un pod, incluyendo las inyectadas
kubectl exec -it consumidor -- env | grep SERVICE

# Resultado tipico:
# API_BACKEND_SERVICE_HOST=10.96.45.12
# API_BACKEND_SERVICE_PORT=8080
# KUBERNETES_SERVICE_HOST=10.96.0.1
# KUBERNETES_SERVICE_PORT=443
```

> Las variables de entorno tienen una limitacion importante: solo incluyen servicios que existian al momento de la creacion del pod. Si un servicio nuevo se crea despues, el pod no recibira la variable. El DNS interno no tiene esta limitacion y es el mecanismo preferido para service discovery.

## Registros centrales: Consul, etcd, ZooKeeper

Mas alla del mecanismo nativo del orquestador, existen registros centrales de servicio que operan como sistemas distribuidos de clave-valor. Consul, etcd y ZooKeeper son los mas utilizados, y cada uno ofrece un modelo diferente de service discovery con distintas garantias de consistencia.

```yaml
# Ejemplo de registro en Consul con health checks
services:
  - name: api-backend
    id: api-backend-1
    address: 10.0.1.15
    port: 8080
    tags:
      - produccion
      - v2
    check:
      http: http://10.0.1.15:8080/health
      interval: 10s
      timeout: 5s
```

```bash
# Registrar un servicio manualmente en Consul
curl -X PUT http://consul:8500/v1/agent/service/register \
  -d '{
    "Name": "api-backend",
    "Address": "10.0.1.15",
    "Port": 8080,
    "Tags": ["produccion"]
  }'

# Consultar servicios registrados
curl -s http://consul:8500/v1/catalog/service/api-backend | jq .

# Usar Consul DNS para resolver servicios
dig @127.0.0.1 -p 8600 api-backend.service.consul SRV
```

```bash
# etcd: registrar un servicio como clave-valor
etcdctl put /services/api-backend/endpoints/10.0.1.15:8080

# etcd: consultar los endpoints registrados
etcdctl get /services/api-backend/endpoints --prefix

# ZooKeeper: crear un nodo efimero para registrar instancia
zkCli.sh create -e /services/api-backend/instance-1 \
  '{"host":"10.0.1.15","port":8080}'
```

> Los registros centrales como Consul ofrecen funcionalidades que el DNS basico del orquestador no provee: health checks activos, observacion de cambios en tiempo real (watch), multi-datacenter y cifrado interno (mTLS). Son especialmente utiles en entornos multi-clúster o hibridos.

## Patron DNS en aplicaciones

Las aplicaciones modernas disenadas para entornos de contenedores utilizan los nombres DNS del servicio como punto de contacto unico. En lugar de configurar direcciones IP en archivos de configuracion, la aplicacion resuelve el nombre del servicio cada vez que necesita conectarse. Este patron elimina la necesidad de reconfigurar la aplicacion ante cambios en la infraestructura.

```python
# Ejemplo en Python: conexion a un servicio usando DNS del clúster
import socket

def resolver_servicio(nombre, namespace="default"):
    fqdn = f"{nombre}.{namespace}.svc.cluster.local"
    try:
        ip = socket.gethostbyname(fqdn)
        return ip
    except socket.gaierror:
        return None

# Uso
ip_api = resolver_servicio("api-backend", "produccion")
print(f"API Backend IP: {ip_api}")
```

```yaml
# Ejemplo de configuracion que usa nombres DNS internos
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.produccion.svc.cluster.local"
  DATABASE_PORT: "5432"
  CACHE_HOST: "redis.cache.svc.cluster.local"
  CACHE_PORT: "6379"
```

El service discovery basado en DNS es el patron mas adoptado porque es estandar, interoperable y funciona con cualquier lenguaje o framework sin dependencias especificas del orquestador. Las librerias de cliente de bases de datos y mensajeria modernas soportan nativamente la resolucion de nombres dinamica, lo que hace que este patron sea transparente para el desarrollador.

