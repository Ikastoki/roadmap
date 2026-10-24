# Actuator

Spring Boot Actuator proporciona capacidades de monitoreo y gestión de aplicaciones Spring Boot mediante endpoints HTTP o JMX. Estos endpoints exponen información sobre el estado de la aplicación, métricas de rendimiento, configuración y salud, siendo fundamentales para operaciones en producción.

Los endpoints más comunes incluyen `/health` que muestra el estado general de la aplicación y sus dependencias, `/info` que expone información personalizada del proyecto, `/metrics` que proporciona métricas de rendimiento como uso de memoria y tiempos de respuesta, y `/env` que muestra las propiedades de entorno y configuración.

```yaml
# Configuración básica de Actuator en application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,beans,mappings
  endpoint:
    health:
      show-details: when_authorized
  server:
    port: 8081
```

La exposición de endpoints requiere configuración explícita por razones de seguridad. Por defecto, solo los endpoints `/health` e `/info` están expuestos. Para habilitar otros endpoints, se debe configurar `management.endpoints.web.exposure.include`. Se puede usar `*` para exponer todos los endpoints, pero esto no es recomendado en producción por razones de seguridad.

```properties
# application.properties
management.endpoints.web.exposure.include=health,info,metrics,env,beans,mappings
management.endpoints.web.base-path=/manage
management.server.port=8081
```

La seguridad de los endpoints es crítica. Spring Boot Actuator se integra automáticamente con Spring Security si está presente en el classpath. La configuración por defecto requiere autenticación para todos los endpoints excepto `/health` cuando se configura con `show-details: when_authorized`.

```java
@Configuration
public class ActuatorSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .requestMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                .anyRequest().hasRole("ADMIN")
            )
            .httpBasic(withDefaults());
        
        return http.build();
    }
}
```

Los health indicators permiten monitorear la salud de dependencias externas como bases de datos, sistemas de mensajería y servicios web. Spring Boot incluye health indicators para muchos componentes comunes. Cuando una dependencia no está disponible, su health indicator reporta DOWN, afectando el estado general de la aplicación.

```java
@Component
public class ApiExternaHealthIndicator implements HealthIndicator {
    
    private final RestTemplate restTemplate;
    private final String apiUrl;
    
    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                apiUrl + "/health", String.class);
            
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("apiUrl", apiUrl)
                    .withDetail("statusCode", response.getStatusCode().value())
                    .build();
            } else {
                return Health.down()
                    .withDetail("apiUrl", apiUrl)
                    .withDetail("statusCode", response.getStatusCode().value())
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("apiUrl", apiUrl)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

Micrometer es el framework de métricas que integra Actuator con múltiples sistemas de monitoreo como Prometheus, Datadog, New Relic y Atlas. Micrometer proporciona una API unificada para registrar métricas que luego se exportan al sistema de monitoreo configurado. Esto permite cambiar sistemas de monitoreo sin modificar el código de la aplicación.

```java
@Service
public class PedidoService {
    
    private final MeterRegistry meterRegistry;
    private final Timer.Sample sample;
    
    public PedidoService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.sample = Timer.start(meterRegistry);
    }
    
    public void procesarPedido(Pedido pedido) {
        // Incrementar contador
        meterRegistry.counter("pedidos.procesados.total").increment();
        
        // Registrar tiempo de procesamiento
        Timer.builder("pedidos.procesamiento.tiempo")
            .description("Tiempo de procesamiento de pedidos")
            .register(meterRegistry)
            .record(() -> {
                // Lógica de procesamiento
            });
        
        // Gauge personalizado
        Gauge.builder("pedidos.cola.tamanio", this::getColaTamanio)
            .description("Tamaño de la cola de pedidos")
            .register(meterRegistry);
    }
}
```

Los audit events permiten registrar eventos de seguridad importantes como intentos de login exitosos y fallidos. Spring Boot Actuator registra automáticamente eventos de autenticación cuando Spring Security está presente. Estos eventos se pueden personalizar y consultar a través del endpoint `/auditevents`.

```java
@Component
public class CustomAuditEventRepository implements AuditEventRepository {
    
    private final List<AuditEvent> events = new ArrayList<>();
    
    @Override
    public void add(AuditEvent event) {
        events.add(event);
        // Aquí se puede agregar lógica adicional como persistencia
    }
    
    @Override
    public List<AuditEvent> find(String principal, Instant after, Instant before) {
        return events.stream()
            .filter(e -> e.getPrincipal().equals(principal))
            .filter(e -> e.getTimestamp().isAfter(after))
            .filter(e -> e.getTimestamp().isBefore(before))
            .collect(Collectors.toList());
    }
}
```

> Los endpoints de Actuator exponen información sensible de la aplicación. En producción, se debe configurar cuidadosamente qué endpoints están expuestos y bajo qué condiciones de autenticación. No se recomienda exponer endpoints como `/env` o `/beans` sin autenticación robusta.

