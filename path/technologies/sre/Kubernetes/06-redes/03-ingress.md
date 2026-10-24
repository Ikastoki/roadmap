# Ingress

Un Service de tipo LoadBalancer expone un servicio al exterior, pero cada servicio requiere su propio balanceador de carga, cada uno con su propia IP y su propio coste economico. En una arquitectura de microservicios con una docena de servicios, este modelo es insostenible. Ingres resuelve este problema proporcionando un unico punto de entrada HTTP/HTTPS al cluster, capaz de enrutar peticiones a diferentes servicios basandose en el nombre de host, la ruta de la URL o ambos.

Ingress no es un balanceador de carga al uso. Es un recurso declarativo que define reglas de enrutamiento de capa 7 (aplicacion). La implementacion real del trafico corre a cargo de un controlador de Ingress, que es un proxy inverso (NGINX, Traefik, HAProxy, Envoy) configurado y gestionado por Kubernetes. El recurso Ingress dice *que* debe ocurrir; el controlador dice *como* hacerlo.

## El recurso Ingress

Un Ingress se define como cualquier otro recurso de Kubernetes mediante un manifiesto YAML. La especificacion contiene las reglas de enrutamiento, los certificados TLS y, opcionalmente, configuraciones adicionales a traves de anotaciones.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-principal
spec:
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-svc
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-svc
                port:
                  number: 80
    - host: frontend.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

Cada regla de Ingress puede contener uno o varios paths, y cada path redirige el trafico a un Service diferente. El `pathType` admite tres valores:

- `Prefix`: coincide si la ruta de la peticion comienza con el prefijo indicado.
- `Exact`: coincide solo si la ruta es identica.
- `ImplementationSpecific`: el comportamiento depende del controlador (tipicamente similar a Prefix).

Las reglas se evaluan en orden de especificidad. Si una peticion coincide con varias reglas, gana la mas especifica. Por ejemplo, `/v1/usuarios` coincide con el prefijo `/v1`, pero si existiera una regla exacta `/v1/usuarios`, esta tendria prioridad.

## Controladores de Ingress

El recurso Ingress no hace nada por si mismo. Sin un controlador instalado, el recurso se crea pero el trafico nunca llega a los servicios. El controlador es el componente que lee las reglas del recurso Ingress y configura el proxy inverso correspondiente para implementarlas.

Cada controlador tiene sus particularidades:

- **NGINX Ingress Controller**: el mas extendido. Utiliza NGINX como proxy inverso y se configura mediante anotaciones en el recurso Ingress. Soporta websockets, reescritura de rutas, autenticacion basica, rate limiting, y practicamente cualquier funcion que ofrezca NGINX. Se despliega tipicamente como un Deployment con un Service de tipo LoadBalancer o NodePort.

- **Traefik**: disenado especificamente para entornos de microservicios y contenedores. Descubre automaticamente los servicios sin necesidad de reinicios ni recargas. Incluye un panel de control web, soporte nativo para Let's Encrypt, y middlewares configurables para autenticacion, rate limiting, circuit breakers y mas.

- **HAProxy Ingress**: basado en HAProxy, conocido por su rendimiento y eficiencia de recursos. Ofrece configuracion avanzada de terminos TCP, stickiness de sesiones y metricas detalladas.

- **AWS Load Balancer Controller**: el controlador oficial para AWS. No utiliza un proxy dentro del cluster, sino que provisiona un Application Load Balancer (ALB) en AWS y configura las reglas de enrutamiento directamente en el balanceador de carga externo.

```bash
# Instalar NGINX Ingress Controller con Helm
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Verificar que el controlador esta operativo
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## Terminacion SSL/TLS

Ingress puede gestionar la terminacion de conexiones SSL/TLS, descargando del backend la responsabilidad de manejar certificados. Para ello, el certificado y la clave privada se almacenan en un Secret de tipo `kubernetes.io/tls` y se referencian desde el recurso Ingress.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-midominio
type: kubernetes.io/tls
data:
  tls.crt: <certificado-en-base64>
  tls.key: <clave-privada-en-base64>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-con-tls
spec:
  tls:
    - hosts:
        - api.midominio.com
      secretName: tls-midominio
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

Cuando un Ingress tiene configuracion TLS, el controlador acepta conexiones HTTPS en el puerto 443, descifra el trafico utilizando el certificado del Secret, y reenvia las peticiones en HTTP plano al servicio backend. Esto simplifica la gestion de certificados y permite centralizar la seguridad en el punto de entrada del cluster.

Para entornos de produccion, la integracion con Let's Encrypt a traves de cert-manager automatiza la obtencion y renovacion de certificados. cert-manager observa los recursos Ingress y genera automaticamente los certificados TLS, almacenandolos como Secrets en el namespace correspondiente.

```bash
# Instalar cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Crear un ClusterIssuer para Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@midominio.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

## Anotaciones y configuracion avanzada

Las anotaciones en el recurso Ingress permiten ajustar el comportamiento del controlador sin modificar la estructura base del recurso. Cada controlador define sus propias anotaciones, aunque algunas son comunes en varios controladores.

### Anotaciones tipicas de NGINX Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/cors-enabled: "true"
    nginx.ingress.kubernetes.io/cors-origin: "https://midominio.com"
spec:
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

Las anotaciones mas utilizadas incluyen:

- `rewrite-target`: reescribe la ruta antes de enviarla al backend. Util para exponer servicios con rutas anidadas sin que el backend necesite conocer la ruta base.
- `ssl-redirect`: fuerza la redireccion de HTTP a HTTPS.
- `proxy-body-size`: limita el tamano maximo del cuerpo de la peticion.
- `limit-rps` / `limit-connections`: rate limiting por IP de origen.
- `cors-*`: configuracion de CORS para peticiones desde navegadores.
- `whitelist-source-range`: restringe el acceso a rangos de IP concretos.
- `auth-type` / `auth-secret`: autenticacion basica HTTP.

### Anotaciones en Traefik

Traefik utiliza un sistema de middlewares en lugar de anotaciones directas, aunque tambien soporta anotaciones en el recurso Ingress. Los middlewares son recursos independientes que se referencian desde el Ingress:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    average: 100
    burst: 50
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-rate-limit@kubernetescrd
spec:
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

## Ingress vs Gateway API

El recurso Ingress original tiene limitaciones reconocidas: solo maneja HTTP/HTTPS, mezcla configuracion de infraestructura con configuracion de aplicacion, y no distingue entre el equipo que define el enrutamiento (desarrollador) y el equipo que gestiona la infraestructura (operaciones).

Para superar estas limitaciones, la comunidad de Kubernetes desarrollo Gateway API, un conjunto de recursos (GatewayClass, Gateway, HTTPRoute, TCPRoute, etc.) que separan las responsabilidades y soportan protocolos adicionales (gRPC, TCP, UDP, WebSocket). Gateway API no sustituye a Ingress de inmediato, pero representa el futuro del enrutamiento de red en Kubernetes.

> En clusters actuales, Ingress sigue siendo la opcion mayoritaria y plenamente funcional. Gateway API esta recomendado para proyectos nuevos que requieran caracteristicas avanzadas como分割 de trafico (canary), afinidad de sesion o enrutamiento por encabezados HTTP.

