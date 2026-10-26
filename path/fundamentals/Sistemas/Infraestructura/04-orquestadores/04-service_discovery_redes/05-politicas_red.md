# Politicas de Red

En un clúster de contenedores con red plana, cualquier pod puede comunicarse con cualquier otro pod de forma predeterminada. Esta libertad de comunicacion es conveniente para el desarrollo pero constituye un riesgo de seguridad grave en produccion: si un pod es comprometido, el atacante podria moverse lateralmente hacia cualquier otro componente del clúster sin restricciones. Las politicas de red implementan un firewall a nivel de unidad de trabajo, definiendo con precision que conexiones de entrada (ingress) y salida (egress) estan permitidas.

Las politicas de red operan en el nivel del orquestador, no del sistema operativo. Mientras que iptables o nftables filtran trafico a nivel de nodo, las politicas de red filtran trafico entre pods, basandose en identidad (labels, namespaces) en lugar de direcciones IP. Esta abstraccion es fundamental porque las IP de los pods son efimeras y cambian constantemente, making las reglas basadas en IP inutiles a largo plazo.

## Modelo default-deny vs default-allow

Las politicas de red se implementan en dos modelos opuestos. El modelo `default-deny` bloquea todo el trafico por defecto y solo permite las conexiones explicitamente autorizadas. El modelo `default-allow` permite todo por defecto y solo bloquea lo que se define explicitamente. En entornos de produccion, el modelo default-deny es la practica recomendada porque aplica el principio de minimo privilegio.

```yaml
# Politica default-deny: bloquea todo el trafico de entrada en el namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: produccion
spec:
  podSelector: {}   # Aplica a todos los pods del namespace
  policyTypes:
    - Ingress
```

```yaml
# Politica default-deny para egress: bloquea toda salida
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: produccion
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

```bash
# Verificar las politicas de red aplicadas
kubectl get networkpolicy -n produccion

# NAME                  POD-SELECTOR   AGE
# default-deny-ingress   <none>        5m
# default-deny-egress    <none>        5m

# Probar que el trafico esta bloqueado
kubectl exec -it mi-app -- curl --connect-timeout 3 http://api-backend:8080
# Resultado: connection refused o timeout
```

> Cuando se crea una politica default-deny, todos los pods del namespace quedan incomunicados hasta que se creen politicas adicionales que permitan el trafico necesario. Es recomendable aplicar estas politicas primero en un namespace de prueba antes de implementarlas en produccion.

## Politicas basadas en labels

Las politicas de red en Kubernetes seleccionan los pods afectados y los pods permitidos mediante labels. Esta seleccion por identidad permite definir reglas que se adaptan automaticamente a los cambios en la composicion del clúster. Cuando un nuevo pod con los labels adecuados se despliega, las politicas lo incluyen automaticamente sin necesidad de reconfiguracion.

```yaml
# Permitir trafico de entrada solo desde pods con label app=frontend hacia api-backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: produccion
spec:
  podSelector:
    matchLabels:
      app: api-backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

```yaml
# Permitir que api-backend se conecte a postgres y redis
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-backend-egress
  namespace: produccion
spec:
  podSelector:
    matchLabels:
      app: api-backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53   # DNS: necesario para resolver nombres de servicios
```

```yaml
# Permitir trafico entre namespaces usando namespaceSelector
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: produccion
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoreo
      ports:
        - protocol: TCP
          port: 9090   # Prometheus metrics
```

```bash
# Ver la descripcion detallada de una politica
kubectl describe networkpolicy allow-frontend-to-api -n produccion

# Verificar que la politica funciona
kubectl exec -it mi-app -l app=frontend -- curl --connect-timeout 3 http://api-backend:8080
# Resultado: conexion exitosa

kubectl exec -it mi-app -l app=otro-servicio -- curl --connect-timeout 3 http://api-backend:8080
# Resultado: connection refused (politica bloquea)
```

## Implementacion via plugins CNI

Las NetworkPolicy de Kubernetes son una especificacion que el orquestador interpreta y delega a un plugin CNI compatible. No todos los plugins implementan las politicas de red. Flannel, por ejemplo, no soporta NetworkPolicy. Plugins como Calico, Cilium y Weave si lo hacen, pero cada uno implementa las politicas de forma diferente a nivel de sistema operativo.

```bash
# Calico: implementa NetworkPolicy con iptables y BGP
# Instalar Calico como CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Ver los pods de Calico
kubectl get pods -n kube-system -l k8s-app=calico-node

# Ver las reglas de iptables generadas por Calico
kubectl exec -it -n kube-system calico-node-xxxxx -- iptables -L cali-pool -v -n
```

```bash
# Cilium: implementa NetworkPolicy con eBPF para mayor rendimiento
# Instalar Cilium con el CLI de Cilium
cilium install --version 1.14.0

# Verificar el estado de Cilium
cilium status

# Ver las politicas de red de Cilium (extensiones propias)
cilium policy get

# Cilium提供 observabilidad a nivel de politica
cilium hubble ui
```

```yaml
# Ejemplo de politica avanzada de Cilium (extension del estandar)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-policy
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
                path: "/api/v[12]/.*"
              - method: POST
                path: "/api/v1/login"
```

> Cilium utiliza eBPF (extended Berkeley Packet Filter) para implementar las politicas de red directamente en el kernel de Linux, sin pasar por iptables. Esto ofrece mejor rendimiento y funcionalidades avanzadas como inspeccion de capa 7 (HTTP, gRPC), identidad de pod y observabilidad detallada. Calico es mas maduro y ampliamente utilizado; Cilium es mas innovador pero requiere kernels recientes (>= 4.19).

## Flujo de decision de una politica

Cuando un paquete de red llega a un pod protegido por una NetworkPolicy, el plugin CNI evalua todas las politicas aplicables en orden. Si al menos una politica permite la conexion, el paquete es aceptado. Si existe alguna politica que seleccione al pod pero no permite la conexion, el paquete es rechazado. Si no hay ninguna politica que seleccione al pod, se aplica el comportamiento por defecto del plugin (permitir o bloquear segun la configuracion del clúster).

```
Paquete entrante a pod X
    │
    ▼
┌─────────────────────────┐
│ Existe alguna policy    │──── No ──▶ Comportamiento por defecto
│ que seleccione al pod X?│            (permitir en mayoria de CNIs)
└─────────┬───────────────┘
          │ Si
          ▼
┌─────────────────────────┐
│ Alguna de las policies  │──── Si ──▶ Paquete permitido
│ permite esta conexion?  │
└─────────┬───────────────┘
          │ No
          ▼
      Paquete rechazado
```

> Las NetworkPolicy se evalúan de forma acumulativa: si existe al menos una politica que permita la conexion, el trafico pasa. No hay necesidad de que una unica politica incluya todas las reglas; se pueden crear multiples politicas que se complementen entre si.

