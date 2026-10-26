# Seguridad de Red

La red dentro de un clúster de Kubernetes es inherentemente abierta: por defecto, cualquier pod puede comunicarse con cualquier otro pod sin restricciones. Esta conectividad total facilita el desarrollo, pero en producción representa un riesgo significante si un componente comprometido puede propagarse lateralmente hacia servicios críticos. La seguridad de red en Kubernetes abarca tres dimensiones: el cifrado del tráfico entre componentes, la segmentación mediante politicas de red y la orquestación avanzada mediante service meshes.

## Cifrado con mTLS

El tráfico dentro del clúster viaja en texto plano por defecto. Incluso cuando se utiliza HTTPS hacia el exterior, la comunicación entre pods internos no está cifrada. mTLS (Mutual TLS) establece cifrado y autenticación bidireccional entre servicios: cada parte verifica la identidad de la otra mediante certificados, y el tráfico se cifra con claves efímeras generadas por sesión. Esto impide la interceptación y la suplantación de identidad entre componentes.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mtls-global
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

> El modo `STRICT` rechaza cualquier conexión que no utilice mTLS. El modo `PERMISSIVE` acepta tanto conexiones cifradas como en texto plano, lo cual es útil durante la migración pero no recomendado en producción.

```yaml
apiVersion: security.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-backend
  namespace: produccion
spec:
  host: api-backend.produccion.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: gateway-externo
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: certificado-externo
      hosts:
        - "api.empresa.com"
```

> Istio genera y gestiona automáticamente los certificados de mTLS para todos los proxies sidecar. Los certificados de corta duración (24 horas por defecto) se rotan sin intervención manual, reduciendo la ventana de exposición si una clave se ve comprometida.

## Politicas de red

Las politicas de red de Kubernetes actúan como firewalls a nivel de pod. Permiten definir reglas de entrada (ingress) y salida (egress) que determinan qué tráfico está permitido hacia y desde un conjunto de pods seleccionados por etiquetas. Sin una politica de red implementada, todos los pods tienen acceso completo a todos los demás.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: politica-api-backend
  namespace: produccion
spec:
  podSelector:
    matchLabels:
      app: api-backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              environment: produccion
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 53
        - protocol: UDP
          port: 53
```

> Las Network Policies son implementadas por los plugins de red (CNI). No todos los proveedores las soportan. Calico, Cilium y Weave Net son populares por su soporte completo. Flannel, por ejemplo, no implementa Network Policies de forma nativa.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-default
  namespace: produccion
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

> Una politica con `podSelector: {}` (selector vacío) se aplica a todos los pods del namespace. Una政治 de denegación por defecto es la base sobre la cual se construyen reglas más específicas. Sin esta base, las reglas de permitir no tienen efecto real porque la ausencia de politica implica acceso total.

## Service Mesh

Un service mesh como Istio o Linkerd añade una capa de infraestructura de red dedicada a la comunicación entre servicios. Opera mediante proxies sidecar que se inyectan automáticamente en cada pod, interceptando todo el tráfico de red. Estos proxies gestionan el cifrado, la autenticación, el balanceo de carga, la observabilidad y la implementación de politicas de tráfico de forma transparente para las aplicaciones.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-backend-authz
  namespace: produccion
spec:
  selector:
    matchLabels:
      app: api-backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/produccion/sa/frontend-sa"
              - "cluster.local/ns/produccion/sa/gateway-sa"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

> Istio distingue entre `ALLOW`, `DENY` y `CUSTOM` como acciones. Una `DENY` explícita tiene prioridad sobre una `ALLOW` en el orden de evaluación. Esto permite crear reglas de denegación específica que sobreescriban politicas de permiso general.

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: produccion
spec:
  selector:
    matchLabels:
      app: api-backend
  jwtRules:
    - issuer: "https://auth.empresa.com"
      jwksUri: "https://auth.empresa.com/.well-known/jwks.json"
      audiences:
        - "api.empresa.com"
```

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-backend
  namespace: produccion
spec:
  hosts:
    - api-backend
  http:
    - route:
        - destination:
            host: api-backend
            subset: v2
          weight: 90
        - destination:
            host: api-backend
            subset: v1
          weight: 10
```

## Segmentacion avanzada con Cilium

Cilium es un plugin de red CNI basado en eBPF que ofrece capacidades avanzadas de seguridad de red a nivel de kernel. A diferencia de las Network Policies estándar de Kubernetes, Cilium puede inspeccionar tráfico a nivel de applicación (HTTP, gRPC, Kafka) y aplicar politicas basadas en identidades de servicio en lugar de direcciones IP.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: politica-http-segura
  namespace: produccion
spec:
  endpointSelector:
    matchLabels:
      app: api-backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "/api/.*"
              - method: POST
                path: "/api/v2/.*"
  egress:
    - toEndpoints:
        - matchLabels:
            app: database
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

> Las CiliumNetwork Policies son un superconjunto de las Network Policies estándar de Kubernetes. Si una política específica de Cilium no coincide, las políticas estándar de Kubernetes siguen aplicándose. Esto permite una migración gradual desde las políticas básicas hacia las avanzadas.

