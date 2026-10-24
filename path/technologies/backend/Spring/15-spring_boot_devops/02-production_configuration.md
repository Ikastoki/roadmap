# Production Configuration

Una aplicacion Spring Boot en produccion requiere configuraciones especificas que van mas alla de las propiedades basicas del servidor y base de datos. La gestion de pools de conexiones, hilos, cache y mecanismos de resiliencia determina directamente la estabilidad, el rendimiento y la capacidad de recuperacion ante fallos del sistema.

## Connection pooling con HikariCP

HikariCP es el pool de conexiones por defecto en Spring Boot y uno de los mas eficientes en el ecosistema Java. Su configuracion correcta es critica para evitar agotamiento de conexiones bajo carga y garantizar tiempos de respuesta estables.

```yaml
spring:
  datasource:
    hikari:
      pool-name: ProductionPool
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 600000
      connection-timeout: 10000
      validation-timeout: 5000
      leak-detection-threshold: 60000
      connection-test-query: SELECT 1
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
```

El `maximum-pool-size` debe ajustarse segun la capacidad de la base de datos y el numero de instancias de la aplicacion. Una regla comun es `pool_size = (core_count * 2) + effective_spindle_count`, pero en entornos cloud con bases de datos gestionadas, 10-20 conexiones por instancia suelen ser suficientes.

> `leak-detection-threshold` registra un warning cuando una conexion permanece tomada mas de 60 segundos, lo que permite detectar fugas de conexiones en codigo que no cierra los recursos correctamente.

## Thread pool

La configuracion del pool de hilos afecta directamente la capacidad de procesamiento paralelo de la aplicacion. Spring Boot ofrece `ThreadPoolTaskExecutor` para gestionar hilos de forma centralizada.

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 8
        max-size: 32
        queue-capacity: 100
        keep-alive: 60s
        allow-core-thread-timeout: true
      thread-name-prefix: async-
    scheduling:
      pool:
        size: 4
      thread-name-prefix: scheduler-
```

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(32);
        executor.setQueueCapacity(100);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

## Cache

Spring Cache abstraction permite integrar multiples proveedores de cache (Ehcache, Redis, Caffeine) con una API unica. En produccion, la configuracion del cache debe equilibrar frescura de datos con reduccion de carga en la base de datos.

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 3600000
      cache-null-values: false
      key-prefix: "app:"
      use-key-prefix: true
```

```java
@Service
public class ProductoService {

    @Cacheable(value = "productos", key = "#id", unless = "#result == null")
    public Producto obtenerPorId(Long id) {
        return productoRepository.findById(id).orElse(null);
    }

    @CachePut(value = "productos", key = "#producto.id")
    public Producto actualizar(Producto producto) {
        return productoRepository.save(producto);
    }

    @CacheEvict(value = "productos", key = "#id")
    public void eliminar(Long id) {
        productoRepository.deleteById(id);
    }

    @CacheEvict(value = "productos", allEntries = true)
    @Scheduled(fixedDelay = 3600000)
    public void limpiarCache() {
        // Limpieza periodica del cache
    }
}
```

## Resiliencia

Spring Cloud Circuit Breaker y Resilience4j proporcionan mecanismos de resiliencia para proteger la aplicacion contra fallos en servicios externos y propagacion de errores en cascada.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      servicioExterno:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
  retry:
    instances:
      servicioExterno:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
  timelimiter:
    instances:
      servicioExterno:
        timeoutDuration: 5s
```

```java
@Service
public class ServicioExternoClient {

    @CircuitBreaker(name = "servicioExterno", fallbackMethod = "fallback")
    @Retry(name = "servicioExterno")
    @TimeLimiter(name = "servicioExterno")
    public CompletableFuture<RespuestaDTO> llamarApiExterna(SolicitudDTO solicitud) {
        return CompletableFuture.supplyAsync(() ->
            webClient.post()
                .uri("/api/externa")
                .bodyValue(solicitud)
                .retrieve()
                .bodyToMono(RespuestaDTO.class)
                .block()
        );
    }

    public CompletableFuture<RespuestaDTO> fallback(SolicitudDTO solicitud, Throwable t) {
        return CompletableFuture.completedFuture(
            RespuestaDTO.fallback("Servicio no disponible")
        );
    }
}
```

La configuracion de produccion debe validarse exhaustivamente antes del despliegue. Utilizar `spring-boot-configuration-processor` para generar metadatos de autocompletado y `@ConfigurationProperties` con `@Validated` para detectar errores de configuracion durante el arranque, no en tiempo de ejecucion.

