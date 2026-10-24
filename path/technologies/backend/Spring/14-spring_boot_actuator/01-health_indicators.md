# Health Indicators

Spring Boot Actuator proporciona un sistema de verificacion de salud que permite a los orquestadores de contenedores, load balancers y herramientas de monitorizacion determinar si una aplicacion esta operativa y puede recibir trafico. Los health indicators son componentes que evaluan el estado de diferentes partes del sistema y reportan un consolidado a traves del endpoint `/actuator/health`.

## Interfaz HealthIndicator

Cada indicador de salud implementa la interfaz `HealthIndicator`, que define un unico metodo `health()` que retorna un objeto `Health` con el estado resultante. El objeto `Health` contiene un `Status` (UP, DOWN, OUT_OF_SERVICE o CUSTOM), un mapa de detalles adicionales y un constructor que permite adjuntar informacion diagnostica.

```java
@Component
public class BaseDatosHealthIndicator implements HealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    public BaseDatosHealthIndicator(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Health health() {
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            return Health.up()
                .withDetail("database", "disponible")
                .withDetail("pool", obtenerEstadisticasPool())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }

    private Map<String, Object> obtenerEstadisticasPool() {
        HikariDataSource ds = (HikariDataSource) jdbcTemplate.getDataSource();
        Map<String, Object> stats = new HashMap<>();
        stats.put("activas", ds.getHikariPoolMXBean().getActiveConnections());
        stats.put("disponibles", ds.getHikariPoolMXBean().getIdleConnections());
        stats.put("totales", ds.getHikariPoolMXBean().getTotalConnections());
        return stats;
    }
}
```

Spring Boot autoconfigura health indicators para la mayoria de los componentes conectados: bases de datos (`DataSourceHealthIndicator`), Redis (`RedisHealthIndicator`), MongoDB (`MongoHealthIndicator`), RabbitMQ (`RabbitHealthIndicator`), entre otros. La configuracion de estos se realiza en `application.yml`.

```yaml
management:
  endpoint:
    health:
      show-details: when_authorized
  health:
    db:
      enabled: true
    diskspace:
      enabled: true
    ldap:
      enabled: true
```

## Anotacion @HealthIndicator

Spring Boot permite crear health indicators de forma simplificada utilizando la anotacion `@HealthIndicator` sobre un componente que retorne un `Health` o un `Status`. Esta anotacion elimina la necesidad de implementar la interfaz manualmente.

```java
@HealthIndicator
@Component
public class ServicioExternoHealthIndicator {

    private final WebClient webClient;

    public ServicioExternoHealthIndicator(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("https://api.externa.com").build();
    }

    public Health health() {
        try {
            Mono<ResponseEntity<Void>> respuesta = webClient.get()
                .uri("/ping")
                .retrieve()
                .toBodilessEntity()
                .timeout(Duration.ofSeconds(3))
                .onErrorResume(e -> Mono.just(ResponseEntity.status(503).build()));

            ResponseEntity<Void> response = respuesta.block();
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up().withDetail("api", "respondiendo").build();
            }
            return Health.down().withDetail("api", "no disponible").build();
        } catch (Exception e) {
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```

> En aplicaciones WebFlux, los health indicators deben retornar `Mono<Health>` en lugar de `Health` directamente para mantener la no-bloqueacion. Spring detecta automaticamente el tipo de retorno y gestiona la suscripcion.

## Estados de salud

El sistema de estados de `Health` soporta cuatro valores estandar y permite crear estados personalizados:

- **UP**: la aplicacion o componente esta funcionando correctamente y puede recibir trafico
- **DOWN**: la aplicacion o componente no esta disponible o no puede procesar peticiones
- **OUT_OF_SERVICE**: la aplicacion esta disponible pero no deberia recibir trafico (ej: mantenimiento planificado)
- **UNKNOWN**: el estado no puede determinarse

```java
@HealthIndicator
@Component
public class MantenimientoHealthIndicator {

    @Value("${app.mantenimiento:false}")
    private boolean enMantenimiento;

    public Health health() {
        if (enMantenimiento) {
            return Health.outOfService()
                .withDetail("razon", "Mantenimiento programado")
                .withDetail("until", "2026-01-01T00:00:00Z")
                .build();
        }
        return Health.up().build();
    }
}
```

## Grupos de salud

Spring Boot 2.6+ introduce el concepto de **health groups** que permiten agrupar indicadores segun su proposito. Los grupos mas comunes son `readiness` (indica si la aplicacion puede recibir trafico) y `liveness` (indica si la aplicacion esta viva y no necesita reinicio).

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: db,redis,servicioExterno
          show-details: always
        liveness:
          include: diskspace,process
          show-details: never
      roles: ACTUATOR
```

```bash
# Consultar estado completo
curl http://localhost:8080/actuator/health

# Consultar grupo de readiness
curl http://localhost:8080/actuator/health/readiness

# Consultar grupo de liveness
curl http://localhost:8080/actuator/health/liveness
```

Los grupos de salud son especialmente relevantes en Kubernetes, que distingue entre liveness probes (reiniciar el contenedor si falla) y readiness probes (dejar de enviar trafico si falla). Spring Boot expone los grupos como endpoints independientes que Kubernetes puede consultar directamente.

Los detalles de cada indicador se muestran segun la politica de `show-details`: `never` (nunca), `always` (siempre) o `when_authorized` (solo para usuarios autenticados con rol `ACTUATOR`). En entornos de produccion, se recomienda `when_authorized` para evitar filtrar informacion sensible sobre la infraestructura interna.

