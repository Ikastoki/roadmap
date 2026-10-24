# Network Policies

El modelo de red plano de Kubernetes tiene una consecuencia que puede resultar contraintuitiva: por defecto, cualquier pod puede comunicarse con cualquier otro pod del cluster, sin restricciones. Esto es deseable para la flexibilidad del desarrollo, pero inaceptable para entornos donde coexisten aplicaciones con distintos niveles de confianza, como un frontend web que debe hablar solo con su API, y esta unicamente con su base de datos.

Las Network Policies son el mecanismo que introduce control de trafico a nivel de pod. Funcionan como un firewall interno del cluster: definen reglas de ingreso (quien puede enviar trafico a un pod) y de egreso (a donde puede enviar trafico un pod). No son reglas de red tradicionales basadas en IPs de origen y destino, sino que operan sobre etiquetas (labels), lo que permite politicas dinamicas que se adaptan automaticamente cuando los pods se crean, destruyen o reetiquetan.

## Politica por defecto: allow-all

Cuando no hay Network Policies en un namespace, todo el trafico esta permitido. Un pod en el namespace `produccion` puede comunicarse con cualquier pod en `staging`, cualquier servicio en `kube-system`, o incluso con pods en otros namespaces sin restriccion. Esta es la politica por defecto de Kubernetes, y es intencionada: no asumir ningun aislamiento hasta que se declare explicitamente.

El momento en que se aplica una Network Policy que afecta a un pod, el comportamiento cambia drasticamente. Kubernetes pasa de un modelo implicto de allow-all a un modelo explicito de denegacion por defecto. Cualquier trafico que no este permitido explicitamente por alguna Network Policy sera bloqueado.

> Esta transicion de permitir todo a denegar todo ocurre en el momento exacto en que la primera Network Policy selecciona al pod. No hay periodo de gracia ni modo compatibilidad. Por eso es recomendable aplicar primero una politica que permita el trafico minimo necesario y luego ir anadiendo restricciones.

## Estructura de una NetworkPolicy

Una NetworkPolicy se define como un recurso de Kubernetes en el grupo `networking.k8s.io/v1`. Su especificacion contiene tres elementos principales: un `podSelector` que determina a que pods se aplica la politica, y las reglas de `ingress` y `egress` que definen el trafico permitido.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: produccion
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: base-datos
      ports:
        - port: 5432
          protocol: TCP
```

El campo `policyTypes` indica que tipos de trafico se controlan. Si solo se lista `Ingress`, las reglas de egreso no se modifican (siguen permitiendo todo). Si solo se lista `Egress`, ocurre lo contrario. Si se listan ambos, cualquier trafico que no coincida con las reglas sera bloqueado.

## Selectores de pods y namespaces

Las Network Policies utilizan selectores en lugar de direcciones IP. Esto las hace inherentemente dinamicas: cuando se crea un nuevo pod con la etiqueta `app: frontend`, automaticamente queda cubierto por las politicas que seleccionan esa etiqueta, sin necesidad de actualizar reglas.

El `podSelector` dentro de `from` o `to` selecciona pods dentro del mismo namespace que la NetworkPolicy. Para seleccionar pods de otros namespaces, se utiliza `namespaceSelector`:

```yaml
spec:
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              environment: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - port: 9090
          protocol: TCP
```

En este ejemplo, solo los pods con la etiqueta `app: prometheus` que esten en namespaces con la etiqueta `environment: monitoring` pueden acceder al puerto 9090 de los pods seleccionados.

El `namespaceSelector` puede combinarse con `podSelector` en el mismo bloque. Cuando ambos aparecen, la condicion es AND: deben cumplirse ambas. Si aparecen en bloques separados dentro de `from`, la condicion es OR: cualquiera de los bloques puede coincidir.

```yaml
spec:
  ingress:
    - from:
        # Bloque 1: pods con app=frontend en cualquier namespace
        - podSelector:
            matchLabels:
              app: frontend
        # Bloque 2: cualquier pod del namespace monitoring
        - namespaceSelector:
            matchLabels:
              environment: monitoring
```

## Reglas de ingreso (Ingress)

Las reglas de ingreso controlan el trafico entrante a los pods seleccionados. Cada regla especifica un conjunto de origenes (`from`) y un conjunto de puertos. El trafico que no coincida con ningun origen ni puerto permitido es bloqueado.

El campo `from` acepta varios tipos de selectores:

- `podSelector`: pods del mismo namespace que coinciden con las etiquetas.
- `namespaceSelector`: todos los pods de namespaces que coinciden con las etiquetas.
- `ipBlock`: un rango CIDR externo al cluster (para trafico que viene de fuera).

`ipBlock` permite controlar trafico que no tiene etiquetas, como el que llega de balanceadores de carga externos, VPNs o rangos de IPs de oficinas corporativas.

```yaml
spec:
  ingress:
    - from:
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.96.0.0/12  # Excluir IPs de servicios ClusterIP
      ports:
        - port: 443
          protocol: TCP
```

## Reglas de egreso (Egress)

Las reglas de egreso controlan el trafico saliente de los pods seleccionados. Funcionan de forma simetrica a las de ingreso, pero con un campo `to` en lugar de `from`.

```yaml
spec:
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - port: 8080
          protocol: TCP
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - port: 53
          protocol: UDP
```

Un patron comun es permitir la salida a Internet solo para rangos de IPs especificos (DNS, repositorios de paquetes, APIs externas) y bloquear el resto. Esto evita que un pod comprometido pueda establecer comunicacion con servidores de comando y control.

## Aislamiento de namespaces

Una de las aplicaciones mas utiles de las Network Policies es aislar namespaces completos. La combinacion de `namespaceSelector` y `podSelector` permite crear politicas como "el namespace de produccion solo acepta trafico del namespace de frontend" o "el namespace de base de datos no permite ninguna conexion entrante desde fuera".

Para aislar completamente un namespace, se aplica una politica que deniega todo el trafico entrante (y opcionalmente el saliente) seleccionando todos los pods del namespace con un `podSelector` vacio o con `matchLabels: {}`:

```yaml
# Denegar todo el trafico entrante a todos los pods del namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: base-datos
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```yaml
# Denegar todo el trafico saliente de todos los pods del namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: base-datos
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

Una vez aplicada la politica de denegacion total, se anaden politicas mas especificas que permitan el trafico necesario. Por ejemplo, permitir que solo los pods de `app: api` accedan a la base de datos:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: base-datos
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              environment: produccion
          podSelector:
            matchLabels:
              app: api
      ports:
        - port: 5432
          protocol: TCP
```

## Requisitos del plugin CNI

Las Network Policies no funcionan con todos los plugins de red. El plugin CNI debe implementar el soporte para Network Policies. Flannel, por ejemplo, no las soporta directamente. Calico, Cilium, Weave Net y Canal (Flannel + Calico) si lo hacen.

```bash
# Verificar si el cluster soporta Network Policies
# Los controladores de red suelen estar en kube-system
kubectl get pods -n kube-system | grep -E 'calico|cilium|weave|canal'

# Con Calico, ver las Network Policies activas
kubectl get networkpolicies --all-namespaces
```

Si se intenta aplicar una NetworkPolicy en un cluster sin un plugin compatible, la politica se crea (el recurso existe) pero no tiene efecto. El trafico seguira fluyendo sin restricciones. No hay una advertencia explicita de que la politica esta siendo ignorada, lo que puede generar una falsa sensacion de seguridad.

## Orden de evaluacion y conflictos

Las Network Policies son aditivas y no exclusivas. Si varias politicas seleccionan al mismo pod, el trafico se permite si **cualquier** politica lo permite. No hay un orden de precedencia ni una jerarquia. Esto significa que si existe una politica que permite el trafico entrante desde cualquier origen, y otra politica que solo permite desde un origen concreto, el resultado es que el trafico entrante esta permitido desde cualquier origen (la politica mas permisiva prevalece).

Para aplicar un modelo de minimo privilegio, la estrategia recomendada es:

1. Aplicar una politica de denegacion total por defecto (`podSelector: {}` con `policyTypes: [Ingress, Egress]`).
2. Anadir politicas especificas que permitan el trafico necesario.

Cualquier politica que se anada despues solo puede anadir permisos, no quitarlos. Si se necesita restringir aun mas, hay que modificar las politicas existentes o eliminar las demasiado permisivas.

```bash
# Ver todas las Network Policies en un namespace
kubectl describe networkpolicies -n produccion

# Ver el efecto combinado de todas las politicas sobre un pod
# (no hay un comando directo, pero se puede inspeccionar cada politica)
kubectl get networkpolicies -n produccion -o yaml
```

> Las Network Policies solo controlan trafico a nivel de pod. No afectan al trafico que pasa por Services, Ingress o kube-proxy. Tampoco bloquean el trafico de sistema como las comprobaciones de health checks del kubelet o las comunicaciones del plano de control. La seguridad de red en Kubernetes requiere una estrategia en capas: Network Policies para el aislamiento interno, Ingress con TLS para el acceso externo, y firewalls perimetrales en los nodos.

