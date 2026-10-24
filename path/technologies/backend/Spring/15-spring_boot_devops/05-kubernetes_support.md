# Kubernetes Support

Spring Cloud Kubernetes integra las aplicaciones Spring Boot con el ecosistema Kubernetes, permitiendo consumir ConfigMaps y Secrets como fuentes de configuracion, descubrir servicios a traves del DNS del cluster y adaptar los health checks al modelo de liveness y readiness probes de Kubernetes. Esta integracion elimina la necesidad de configuraciones manuales y permite que la aplicacion se comporte como un ciudadano nativo de Kubernetes.

## Spring Cloud Kubernetes

El starter `spring-cloud-starter-kubernetes-fabric8-config` (o `spring-cloud-kubernetes-fabric8-config` para versiones recientes) habilita la integracion con Kubernetes. Cuando la aplicacion se ejecuta dentro de un cluster, Spring Cloud Kubernetes detecta automaticamente la API de Kubernetes yutiliza ConfigMaps y Secrets como fuentes de configuracion adicionales.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-config</artifactId>
</dependency>
```

```yaml
spring:
  config:
    import: "optional:kubernetes:"
  cloud:
    kubernetes:
      config:
        enabled: true
        name: mi-aplicacion-config
        namespace: default
        fail-fast: true
      secrets:
        enabled: true
        name: mi-aplicacion-secrets
        namespace: default
        default-enabled: true
```

> La opcion `optional:` antes de `kubernetes:` permite que la aplicacion arranque correctamente fuera de Kubernetes (en local o en CI/CD) sin fallar por la ausencia de la API de Kubernetes.

## ConfigMap

Un ConfigMap en Kubernetes almacena pares clave-valor no sensibles que se pueden inyectar como variables de entorno o archivos de configuracion. Spring Cloud Kubernetes carga automaticamente el ConfigMap indicado y lo integra como fuente de propiedades.

```yaml
# ConfigMap en Kubernetes
apiVersion: v1
kind: ConfigMap
metadata:
  name: mi-aplicacion-config
  namespace: default
data:
  application.yml: |
    app:
      pagination:
        default-size: 20
        max-size: 100
    spring:
      jpa:
        show-sql: false
        hibernate:
          ddl-auto: validate
    logging:
      level:
        com.ejemplo: INFO
```

```yaml
# Deployment en Kubernetes que referencia el ConfigMap
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-aplicacion
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mi-aplicacion
  template:
    metadata:
      labels:
        app: mi-aplicacion
    spec:
      containers:
        - name: mi-aplicacion
          image: mi-registry/mi-aplicacion:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "kubernetes"
          volumeMounts:
            - name: config-volume
              mountPath: /config
              readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: mi-aplicacion-config
```

## Secrets

Los Secrets en Kubernetes almacenan datos sensibles (credenciales, tokens, certificados) en base64. Spring Cloud Kubernetes puede cargar Secrets y exponerlos como propiedades de la aplicacion.

```yaml
# Secret en Kubernetes
apiVersion: v1
kind: Secret
metadata:
  name: mi-aplicacion-secrets
  namespace: default
type: Opaque
data:
  DB_USERNAME: YWRtaW4=  # base64 de "admin"
  DB_PASSWORD: c2VjcmV0bw==  # base64 de "secreto"
  API_KEY: bWlrby1hcGkta2V5
```

```yaml
spring:
  cloud:
    kubernetes:
      secrets:
        enabled: true
        name: mi-aplicacion-secrets
        # Cargar secret como volumen
        volumes:
          enabled: true
          default-mode: 0400
```

## Service Discovery

Kubernetes tiene su propio DNS interno que resuelve los nombres de los servicios dentro del cluster. Spring Cloud Kubernetes traduce las llamadas a servicios de Spring Cloud (`@DiscoveryClient`) a consultas DNS de Kubernetes, eliminando la necesidad de Eureka o Consul cuando se ejecuta en Kubernetes.

```java
@RestController
public class ClienteServicioController {

    private final RestTemplate restTemplate;

    public ClienteServicioController(RestTemplateBuilder builder) {
        // Kubernetes resuelve "servicio-externo" via DNS
        this.restTemplate = builder
            .rootUri("http://servicio-externo:8080")
            .build();
    }

    @GetMapping("/proxy")
    public String proxy() {
        return restTemplate.getForObject("/api/datos", String.class);
    }
}
```

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
        service-name: mi-aplicacion
        namespace: default
```

## Health probes

La configuracion de liveness y readiness probes en Kubernetes determina cuando reiniciar un contenedor y cuando dejar de enviarle trafico. Spring Boot Actuator proporciona los endpoints necesarios que Kubernetes puede consultar.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-aplicacion
spec:
  template:
    spec:
      containers:
        - name: mi-aplicacion
          ports:
            - containerPort: 8080
            - containerPort: 8081  # Actuator
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 20
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8081
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30
```

> `startupProbe` es especialmente relevante para imagenes nativas de GraalVM que arrancan rapidamente, o para aplicaciones JVM con tiempos de arranque largos. Kubernetes no ejecutara liveness/readiness hasta que startupProbe complete exitosamente.

