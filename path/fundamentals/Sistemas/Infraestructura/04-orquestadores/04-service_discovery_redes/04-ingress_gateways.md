# Ingress y Gateways

Cuando multiples servicios dentro del clúster necesitan ser expuestos al exterior a traves de un unico punto de entrada HTTP o HTTPS, el tipo `LoadBalancer` deja de ser escalable: cada servicio crearia su propio balanceador de carga externo, generando costos y complejidad innecesarios. Los recursos Ingress y los API Gateways resuelven este problema proporcionando una capa de proxy reverso que enruta el trafico externo hacia los servicios internos segun reglas de host, path, headers o metodos HTTP.

Un Ingress actua como el cap del telefono del clúster: recibe todas las llamadas externas y las dirige al servicio correcto. Un Gateway, en el contexto de service meshes como Istio, extiende este concepto con funcionalidades avanzadas como politicas de trafico, observabilidad y seguridad a nivel de entrada. Ambos conceptos comparten la idea de un punto de entrada unificado, pero difieren en alcance y complejidad.

## Recursos Ingress en Kubernetes

Un Ingress en Kubernetes es un objeto que define reglas de enrutamiento HTTP/HTTPS hacia servicios internos. Para que el Ingress funcione, se necesita un controlador de Ingress (NGINX, Traefik, HAProxy, etc.) ejecutandose en el clúster que interprete las reglas del recurso y configure el proxy reverso correspondiente.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: produccion
  annotations:
    # Anotaciones especificas del controlador
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.ejemplo.com
        - admin.ejemplo.com
      secretName: tls-api-cert
  rules:
    - host: api.ejemplo.com
      http:
        paths:
          - path: /v1(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-v1
                port:
                  number: 8080
          - path: /v2(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-v2
                port:
                  number: 8080
    - host: admin.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-panel
                port:
                  number: 3000
```

```bash
# Ver los Ingress configurados
kubectl get ingress -n produccion

# NAME          CLASS   HOSTS                ADDRESS         PORTS     AGE
# api-ingress   nginx   api.ejemplo.com      34.102.200.15   80, 443   10m
#                           admin.ejemplo.com

# Ver las reglas detalladas
kubectl describe ingress api-ingress -n produccion

# Verificar que el certificado TLS esta activo
kubectl get secret tls-api-cert -n produccion -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

> El recurso Ingress solo define las reglas de enrutamiento; no implementa el proxy. Sin un controlador de Ingress instalado y funcionando, las reglas no tienen efecto. El controlador es quien realmente configura NGINX, Traefik u otro proxy para que aplique las reglas definidas.

## Enrutamiento por host, path y headers

Las reglas de un Ingress pueden enrutar el trafico segun multiples criterios. El enrutamiento por host permite servir diferentes dominios desde el mismo punto de entrada. El enrutamiento por path permite dirigir rutas distintas hacia servicios diferentes bajo el mismo dominio. Los headers HTTP y los metodos HTTP son criterios adicionales que algunos controladores soportan mediante anotaciones o configuracion avanzada.

```yaml
# Enrutamiento por header personalizado (soportado por NGINX y Traefik)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: header-routing
  namespace: produccion
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      if ($http_x_api_version = "beta") {
        proxy_pass http://api-beta.produccion.svc.cluster.local:8080;
      }
spec:
  ingressClassName: nginx
  rules:
    - host: api.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-stable
                port:
                  number: 8080
```

```yaml
# Enrutamiento por path con prefijo de version
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: versioned-routes
  namespace: produccion
spec:
  ingressClassName: traefik
  rules:
    - host: mi-app.ejemplo.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 8080
          - path: /api/v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 8080
          - path: /static
            pathType: Prefix
            backend:
              service:
                name: cdn-frontend
                port:
                  number: 80
```

```bash
# Probar enrutamiento por host
curl -H "Host: api.ejemplo.com" http://34.102.200.15/v1/users
curl -H "Host: admin.ejemplo.com" http://34.102.200.15/dashboard

# Probar enrutamiento por path
curl https://mi-app.ejemplo.com/api/v1/users
curl https://mi-app.ejemplo.com/api/v2/users
curl https://mi-app.ejemplo.com/static/logo.png
```

## Terminacion TLS centralizada

Un Ingress permite gestionar certificados TLS de forma centralizada en un unico punto. Los certificados se almacenan como Secrets de tipo `kubernetes.io/tls` y se asocian a los hosts del Ingress. El controlador se encarga de realizar el handshake TLS con el cliente y de terminate la conexion, redirigiendo el trafico descifrado hacia los servicios internos en texto plano.

```bash
# Crear un certificado TLS como Secret
kubectl create secret tls tls-api-cert \
  --cert=certificado.crt \
  --key=privada.key \
  -n produccion

# Verificar el Secret
kubectl get secret tls-api-cert -n produccion -o yaml

# Usar cert-manager para gestion automatica de certificados
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

```yaml
# ClusterIssuer de cert-manager para Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@ejemplo.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx

---
# Ingress con cert-manager para emitir certificados automaticamente
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-automatico
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.ejemplo.com
      secretName: api-ejemplo-tls
  rules:
    - host: api.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-backend
                port:
                  number: 8080
```

> cert-manager integra con Let's Encrypt o ACME para emitir y renovar certificados automaticamente. Esto elimina la gestion manual de certificados y garantiza que nunca expiren sin que el operador lo detecte.

## Controladores: NGINX, Traefik, Envoy, Istio Gateway

Existen multiples implementaciones de controladores de Ingress, cada una con fortalezas distintas. NGINX es el mas utilizado por su estabilidad y rendimiento. Traefik se destaca por su integracion nativa con contenedores y su dashboard de configuracion. Envoy es el proxy de alto rendimiento utilizado por service meshes como Istio. Istio Gateway extiende el concepto de Ingress con funcionalidades de service mesh como mTLS, observabilidad y politicas de trafico avanzadas.

```bash
# Instalar el controlador NGINX Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/cloud/deploy.yaml

# Ver los controladores instalados
kubectl get pods -n ingress-nginx

# NAME                                        READY   STATUS
# ingress-nginx-controller-7b4d9c8f4b-xyz   1/1     Running

# Ver el servicio del controlador (recibe el trafico externo)
kubectl get svc -n ingress-nginx

# NAME                       TYPE           EXTERNAL-IP
# ingress-nginx-controller   LoadBalancer   34.102.200.15
```

```yaml
# Istio Gateway: equivalente avanzado del Ingress
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api-gateway
  namespace: produccion
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
        credentialName: api-tls-cert
      hosts:
        - "api.ejemplo.com"
        - "admin.ejemplo.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      tls:
        httpsRedirect: true
      hosts:
        - "api.ejemplo.com"

---
# VirtualService de Istio para enrutamiento avanzado
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-routing
  namespace: produccion
spec:
  hosts:
    - "api.ejemplo.com"
  gateways:
    - api-gateway
  http:
    - match:
        - uri:
            prefix: /v1
      route:
        - destination:
            host: api-v1
            port:
              number: 8080
    - match:
        - uri:
            prefix: /v2
      route:
        - destination:
            host: api-v2
            port:
              number: 8080
      # Canary: 10% del trafico a v2
      - destination:
          host: api-v2-canary
          port:
            number: 8080
        weight: 10
```

> Istio Gateway junto con VirtualService ofrece capacidades que van mas alla del Ingress estandar: traffic splitting para despliegues canary, circuit breaking, retries automaticos, fault injection para testing y enrutamiento basado en headers personalizados. La complejidad operativa es significativamente mayor que un Ingress basico.

