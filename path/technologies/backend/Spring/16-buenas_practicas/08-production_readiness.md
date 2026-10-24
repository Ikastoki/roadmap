# Production Readiness

Una aplicacion esta lista para produccion cuando puede responder a preguntas criticas: esta viva y respondiendo? puede procesar trafico? que pasa cuando se inicia o se apaga? donde estan las credenciales? que pasa si un componente falla? La preparacion para produccion no es una fase final del desarrollo, es un conjunto de practicas que se implementan desde el primer dia y se maduran con cada despliegue.

## Health checks

Los health checks son la base de la disponibilidad en produccion. Permiten a los orquestadores de contenedores (Kubernetes, Docker Swarm) y load balancers determinar automaticamente si una instancia puede recibir trafico.

```yaml
management:
  endpoint:
    health:
      show-details: when_authorized
      group:
        readiness:
          include: db,redis,servicioExterno
        liveness:
          include: diskspace,process
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  server:
    port: 8081
```

```java
@Component
public class ServicioExternoHealthIndicator implements HealthIndicator {

    private final WebClient webClient;

    public ServicioExternoHealthIndicator(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("https://api.externa.com").build();
    }

    @Override
    public Health health() {
        try {
            Mono<ResponseEntity<Void>> respuesta = webClient.get()
                .uri("/ping")
                .retrieve()
                .toBodilessEntity()
                .timeout(Duration.ofSeconds(3))
                .block();

            if (respuesta != null && respuesta.getStatusCode().is2xxSuccessful()) {
                return Health.up().withDetail("api", "disponible").build();
            }
            return Health.down().withDetail("api", "no disponible").build();
        } catch (Exception e) {
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```

> En Kubernetes, la liveness probe determina si el contenedor debe reiniciarse, y la readiness probe determina si debe recibir trafico. Configurar ambas correctamente evita que una instancia fallida siga procesando peticiones y que un reinicio innecesario interrumpa conexiones activas.

## Metrics y monitorizacion

Las metricas proporcionan visibilidad sobre el comportamiento de la aplicacion en tiempo real. Sin metricas, los problemas solo se detectan cuando los usuarios reportan errores.

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${SPRING_PROFILES_ACTIVE:local}
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

```java
@Component
public class MetricasServicio {

    private final Counter pedidosCreados;
    private final Timer tiempoProcesamiento;
    private final DistributionSummary tamanoPedido;

    public MetricasServicio(MeterRegistry registry) {
        this.pedidosCreados = Counter.builder("app.pedidos.creados")
            .description("Numero total de pedidos creados")
            .register(registry);

        this.tiempoProcesamiento = Timer.builder("app.pedidos.tiempo")
            .description("Tiempo de procesamiento de pedidos")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        this.tamanoPedido = DistributionSummary.builder("app.pedidos.items")
            .description("Numero de items por pedido")
            .register(registry);
    }

    public void registrarPedido(PedidoDTO pedido, long tiempoMs) {
        pedidosCreados.increment();
        tiempoProcesamiento.record(tiempoMs, TimeUnit.MILLISECONDS);
        tamanoPedido.record(pedido.getItems().size());
    }
}
```

## Graceful shutdown

El apagado gracioso permite que la aplicacion termine las peticiones en curso antes de cerrarse, evitando perdida de datos o respuestas incompletas a los clientes.

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```java
@Component
@Slf4j
public class ShutdownHandler implements DisposableBean {

    @Override
    public void destroy() {
        log.info("Iniciando apagado gracioso...");
        // Completar operaciones pendientes
        // Cerrar conexiones de base de datos
        // Enviar eventos de shutdown a servicios dependientes
        log.info("Apagado completado");
    }
}
```

## Secrets externos

Las credenciales, tokens y claves de cifrado nunca deben almacenarse en el codigo fuente o en archivos de configuracion versionados. En Kubernetes, se utilizan Secrets nativos. En otros entornos, se pueden usar servicios de gestion de secretos como HashiCorp Vault o AWS Secrets Manager.

```yaml
# En Kubernetes: los secrets se inyectan como variables de entorno
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: db-password
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: api-key
```

```yaml
# Aplicacion lee las variables de entorno
spring:
  datasource:
    password: ${DB_PASSWORD}
app:
  api:
    key: ${API_KEY}
```

## Validacion de configuracion

La configuracion debe validarse durante el arranque para detectar errores antes de que la aplicacion empiece a procesar trafico. `@ConfigurationProperties` con `@Validated` proporciona validacion automatica.

```java
@ConfigurationProperties(prefix = "app.datasource")
@Validated
public class DatasourceProperties {

    @NotBlank(message = "La URL de la base de datos es obligatoria")
    @Pattern(regexp = "^jdbc:(postgresql|mysql|h2)://.*",
             message = "La URL debe ser un JDBC URL valido")
    private String url;

    @NotNull
    @Min(value = 1, message = "El pool debe tener al menos 1 conexion")
    @Max(value = 100, message = "El pool no puede exceder 100 conexiones")
    private Integer maximumPoolSize;

    @NotNull
    @DurationUnit(ChronoUnit.MILLIS)
    @Min(value = 1000, message = "El timeout debe ser al menos 1 segundo")
    private Duration connectionTimeout;
}
```

```java
@Component
@Slf4j
@Validated
public class ConfigValidator implements ApplicationListener<ContextRefreshedEvent> {

    @Value("${app.maintenance.mode:false}")
    private boolean maintenanceMode;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        log.info("Validando configuracion de la aplicacion...");
        if (maintenanceMode) {
            log.warn("Aplicacion en modo mantenimiento - no recibira trafico");
        }
        log.info("Configuracion validada correctamente");
    }
}
```

## Manejo de errores

Los errores deben manejarse de forma consistente, registrar informacion suficiente para diagnostico y devolver respuestas que los clientes puedan procesar sin exponer detalles internos del sistema.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponseDTO> handleNotFound(
            ResourceNotFoundException ex) {
        log.warn("Recurso no encontrado: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponseDTO.of("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponseDTO> handleGeneric(Exception ex) {
        log.error("Error no controlado", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponseDTO.of("INTERNAL_ERROR",
                "Ha ocurrido un error interno. Intente de nuevo."));
    }
}
```

La preparacion para produccion es un checklist continuo: health checks funcionando, metricas exportadas, logs centralizados, secrets externos, graceful shutdown configurado, validacion de configuracion activa y manejo de errores consistente. Cada elemento de esta lista evita un tipo especifico de incidente en produccion.

