# Circuit Breaker

El patron circuit breaker protege los sistemas distribuidos contra fallos en cascada. Cuando un servicio dependiente falla o responde con latencia inaceptable, las peticiones acumuladas agotan los recursos del servicio cliente (hilos, conexiones, memoria), propagando el fallo a otros componentes del sistema. El circuit breaker detecta esta situacion y abre el circuito, rechazando peticiones adicionales de forma inmediata sin intentar alcanzar el servicio caido.

Resilience4j es la libreria de referencia para la implementacion de patrones de resiliencia en el ecosistema de Spring, habiendo reemplazado a Netflix Hystrix (ya en mantenimiento). Ofrece modulos independientes para cada patron: CircuitBreaker, RateLimiter, Retry, Bulkhead y TimeLimiter, que pueden combinarse segun las necesidades de cada componente.

## CircuitBreaker

El CircuitBreaker opera con tres estados: **CLOSED** (operacion normal), **OPEN** (fallo detectado, rechazo inmediato) y **HALF_OPEN** (verificacion de recuperacion). La transicion entre estados se basa en.metricas de fallo: cuando la tasa de fallos supera un umbral configurado, el circuito abre; despues de un periodo de espera, permite un numero limitado de peticiones de prueba; si estas exitan, el circuito se cierra.

```java
@Configuration
public class Resilience4jConfig {

    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .slowCallRateThreshold(80)
                .slowCallDurationThreshold(Duration.ofSeconds(5))
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .permittedNumberOfCallsInHalfOpenState(3)
                .minimumNumberOfCalls(10)
                .slidingWindowSize(10)
                .slidingWindowType(SlidingWindowType.COUNT_BASED)
                .recordExceptions(IOException.class, TimeoutException.class)
                .ignoreExceptions(BusinessException.class)
                .build();

        return CircuitBreakerRegistry.of(config);
    }
}
```

```java
@Service
public class PaymentService {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final PaymentGatewayClient paymentGateway;

    public PaymentService(CircuitBreakerRegistry circuitBreakerRegistry,
                          PaymentGatewayClient paymentGateway) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
        this.paymentGateway = paymentGateway;
    }

    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentGateway.charge(request);
    }

    public PaymentResult paymentFallback(PaymentRequest request, Throwable t) {
        return PaymentResult.pending("Pago en cola, se procesara automaticamente");
    }
}
```

> Los valores de configuracion del CircuitBreaker deben calibrarse segun el comportamiento esperado del servicio dependiente. Un threshold de fallo del 50% es razonable para servicios criticos, mientras que para servicios no esenciales se pueden usar umbrales mas permisivos.

## RateLimiter

El RateLimiter controla la frecuencia con la que un componente puede acceder a un recurso compartido. Limita el numero de llamadas en un periodo de tiempo, protegiendo servicios backend contra picos de trafico que puedan saturar sus recursos.

```java
@Service
public class ExternalApiService {

    @RateLimiter(name = "externalApi", fallbackMethod = "rateLimitFallback")
    public ApiResponse callExternalApi(Request request) {
        return externalApiClient.call(request);
    }

    public ApiResponse rateLimitFallback(Request request, Throwable t) {
        return ApiResponse.cached("Datos no disponibles temporalmente");
    }
}
```

```java
@Bean
public RateLimiterRegistry rateLimiterRegistry() {
    RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(10)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .timeoutDuration(Duration.ofMillis(500))
            .build();

    return RateLimiterRegistry.of(config);
}
```

## Retry

El modulo Retry reintenta una operacion que falla antes de propagar la excepcion. Puede configurarse con un numero maximo de reintentos, un intervalo entre reintentos, un multiplicador para backoff exponencial y condiciones de reintento basadas en el tipo de excepcion.

```java
@Service
public class InventoryClient {

    @Retry(name = "inventoryService", fallbackMethod = "inventoryFallback")
    public Inventory checkStock(String productId) {
        return inventoryServiceClient.getStock(productId);
    }

    public Inventory inventoryFallback(String productId, Throwable t) {
        return Inventory.unknown(productId);
    }
}
```

```java
@Bean
public RetryRegistry retryRegistry() {
    RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .enableExponentialBackoff(1000, 2, 5000)
            .retryExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();

    return RetryRegistry.of(config);
}
```

El backoff exponencial es especialmente util para servicios que experimentan sobrecarga temporal: el primer reintento se ejecuta tras 1 segundo, el segundo tras 2 segundos, y asi sucesivamente, dando tiempo al servicio para recuperarse.

## Bulkhead

El Bulkhead implementa el patron de compartimentacion que limita el numero de llamadas concurrentes a un recurso. Evita que un servicio dependiente lento consuma todos los hilos disponibles en el cliente, manteniendo recursos para atender otras peticiones.

```java
@Service
public class CatalogService {

    @Bulkhead(name = "catalogApi", fallbackMethod = "catalogFallback")
    public Catalog getCatalog(String category) {
        return catalogClient.fetch(category);
    }

    public Catalog catalogFallback(String category, Throwable t) {
        return Catalog.empty(category);
    }
}
```

```java
@Bean
public BulkheadRegistry bulkheadRegistry() {
    BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(25)
            .maxWaitDuration(Duration.ofMillis(500))
            .build();

    return BulkheadRegistry.of(config);
}
```

> El Bulkhead puede implementarse con dos estrategias: semaforo (limita llamadas concurrentes con un semaforo) o thread pool (ejecuta llamadas en un pool dedicado). El semaforo es mas ligero y apropiado para llamadas sincronas, mientras que el thread pool es ideal para operaciones asincronas.

## TimeLimiter

El TimeLimiter impone un limite de tiempo a las operaciones que excedan un umbral configurado. Si la operacion no completa dentro del plazo, se cancela y se lanza una `TimeoutException`. Es especialmente util en combinacion con Retry para evitar que las reintentos se acumulen en operaciones lentas.

```java
@Bean
public TimeLimiterRegistry timeLimiterRegistry() {
    TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(3))
            .cancelRunningFuture(true)
            .build();

    return TimeLimiterRegistry.of(config);
}
```

## Composicion de Patrones

La verdadera potencia de Resilience4j radica en la composicion de multiples patrones. Una operacion puede estar protegida simultaneamente por CircuitBreaker, RateLimiter, Retry, Bulkhead y TimeLimiter, donde cada capa resuelve un problema de resiliencia distinto.

```java
@Service
public class ResilientPaymentService {

    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    @RateLimiter(name = "payment")
    @Bulkhead(name = "payment")
    @Retry(name = "payment")
    @TimeLimiter(name = "payment")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() ->
                paymentGateway.charge(request));
    }

    public CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest request, Throwable t) {
        return CompletableFuture.completedFuture(
                PaymentResult.pending("Pago encolado"));
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        sliding-window-size: 10
  timelimiter:
    instances:
      payment:
        timeout-duration: 3s
  retry:
    instances:
      payment:
        max-attempts: 3
        wait-duration: 500ms
  bulkhead:
    instances:
      payment:
        max-concurrent-calls: 20
```

La combinacion de estos patrones crea un sistema robusto que se degrada de forma controlada ante fallos, evita la sobrecarga de servicios dependientes y se recupera automaticamente cuando las condiciones se normalizan.

