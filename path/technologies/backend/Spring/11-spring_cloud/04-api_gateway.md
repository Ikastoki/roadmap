# API Gateway

Un API Gateway funciona como punto de entrada unico para todas las peticiones externas hacia el ecosistema de microservicios. En lugar de exponer directamente los endpoints de cada微servicio al mundo exterior, el gateway centraliza la autenticacion, el enrutamiento, el rate limiting y la transformacion de peticiones, ocultando la topologia interna del sistema y proporcionando una capa de seguridad y control uniforme.

Spring Cloud Gateway es la implementacion recomendada por Spring, construida sobre Spring Boot 2.x, Project Reactor y Spring Framework 5.x. Utiliza un modelo no bloqueante basado en Netty, lo que le permite manejar un alto volumen de conexiones concurrentes con un consumo de recursos significativamente menor que las soluciones bloqueantes.

## Arquitectura del Gateway

```
Clientes externos
        |
        v
+-------------------+
|  Spring Cloud     |
|  Gateway          |
|  - Routes         |
|  - Predicates     |
|  - Filters        |
+-------------------+
   |      |      |
   v      v      v
svc-A   svc-B   svc-C  (microservicios backend)
```

## Routes

Las rutas son las unidades basicas de configuracion del gateway. Cada ruta define un origen (un predicado que identifica las peticiones que la activan) y un destino (la URL del servicio backend donde se redirige la peticion).

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1

        - id: inventory-service-route
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
          filters:
            - StripPrefix=1

        - id: user-service-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
```

El esquema `lb://` indica que el gateway debe resolver el nombre del servicio a traves del service registry (Eureka, Consul) y balancear la carga entre las instancias disponibles.

## Predicates

Los predicados determinan si una peticion activa una ruta determinada. Spring Cloud Gateway ofrece un conjunto rico de predicados que pueden combinarse para crear reglas de enrutamiento precisas.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: advanced-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
            - Header=X-Api-Version, v2
            - After=2024-01-01T00:00:00+01:00[Europe/Madrid]
            - Cookie=session_id, ^[a-f0-9]{32}$
            - Query=status, pending
          filters:
            - StripPrefix=1
```

Los predicados disponibles incluyen `Path` (ruta URL), `Method` (metodo HTTP), `Header` (cabecera), `Cookie`, `Query` (parametro de query), `After/Before/Between` (temporales), `Host` (cabecera Host) y `RemoteAddr` (direccion IP del cliente).

## Filters

Los filtros interceptan las peticiones antes de que alcancen el servicio backend (pre-filters) o despues de que la respuesta regrese (post-filters). Spring Cloud Gateway incluye filtros estandar para operaciones comunes y permite definir filtros personalizados.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: filtered-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Request-Source, gateway
            - AddRequestParameter=source, gateway
            - AddResponseHeader=X-Response-Time, {T}
            - SetStatus=200
            - name: CircuitBreaker
              args:
                name: orderServiceCircuitBreaker
                fallbackUri: forward:/fallback/orders
```

```java
@Component
public class AuthenticationFilter implements GatewayFilter, Ordered {

    private final TokenValidator tokenValidator;

    public AuthenticationFilter(TokenValidator tokenValidator) {
        this.tokenValidator = tokenValidator;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (token == null || !tokenValidator.isValid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .header("X-User-Id", tokenValidator.extractUserId(token))
                .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }

    @Override
    public int getOrder() {
        return -1;  // ejecutar antes de los filtros estandar
    }
}
```

## Rate Limiting

El rate limiting protege los servicios backend contra sobrecargas al limitar el numero de peticiones que un cliente puede realizar en un periodo de tiempo determinado. Spring Cloud Gateway integra rate limiting basado en Redis utilizando el patron Token Bucket.

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
            key-resolver: "#{@userKeyResolver}"
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
            exchange.getRequest().getHeaders().getFirst("X-User-Id"));
}

// Rate limiting por IP del cliente
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
}
```

> La capacidad de burst (rafaga) debe ser al menos igual a la tasa de replenishment para evitar la perdida de peticiones legitimas durante picos de trafico. Configurar burstCapacity por debajo de replenishRate genera rechazos inesperados.

## Load Balancing

El gateway integra transparentemente el balanceo de carga cuando se utiliza el esquema `lb://` en las URIs de las rutas. Spring Cloud LoadBalancer resuelve el nombre del servicio a traves del service registry y distribuye las peticiones entre las instancias disponibles segun la estrategia configurada.

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 35s
      retry:
        enabled: true
        max-retries-on-same-service-instance: 0
        max-retries-on-next-service-instance: 2
```

## Circuit Breaker Integration

Spring Cloud Gateway se integra con Resilience4j para implementar el patron circuit breaker a nivel de ruta. Cuando un servicio backend falla repetidamente, el circuit breaker abre y las peticiones se redirigen a un fallback configurado en lugar de intentar alcanzar el servicio caido.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: resilient-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: CircuitBreaker
              args:
                name: orderCircuitBreaker
                fallbackUri: forward:/fallback/order-unavailable
                exceptions:
                  - java.util.concurrent.TimeoutException
```

```java
@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @GetMapping("/order-unavailable")
    public ResponseEntity<Map<String, String>> orderUnavailable() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(Map.of(
                        "message", "El servicio de ordenes no esta disponible",
                        "timestamp", Instant.now().toString(),
                        "retryAfterSeconds", "30"));
    }
}
```

> La integracion del circuit breaker en el gateway es la primera linea de defensa contra fallos en cascada. Los servicios backend tambien deben implementar circuit breakers internos para protegerse contra fallos en sus propias dependencias.

La combinacion de rutas, predicados, filtros y mecanismos de resiliencia convierte al API Gateway en una pieza arquitectonica central que controla el trafico entrante, protege los servicios backend y proporciona una API consistente y segura a los clientes externos.

