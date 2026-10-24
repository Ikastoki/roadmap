# Metrics

Spring Boot Actuator integra **Micrometer**, una biblioteca de instrumentacion de aplicaciones que proporciona una facade agnostica para metricas. Micrometer permite colectar contadores, temporizadores y distribuciones de datos de forma estandar, exportandolos a multiples sistemas de monitorizacion como Prometheus, Datadog, New Relic o CloudWatch sin cambiar el codigo de la aplicacion.

## Integracion Micrometer

El starter `spring-boot-starter-actuator` incluye Micrometer Core y la integracion con el `MeterRegistry` de Spring Boot. Cada metrica se registra como un **meter** (medidor) que puede ser de diferentes tipos: `Counter`, `Timer`, `DistributionSummary`, `Gauge` o `LongTaskTimer`.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: mi-aplicacion
      environment: produccion
```

```bash
# Listar todas las metricas disponibles
curl http://localhost:8080/actuator/metrics

# Consultar metrica especifica
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Metricas en formato Prometheus
curl http://localhost:8080/actuator/prometheus
```

## Tipos de metricas

**Counter** es un acumulador que solo incrementa su valor. Es ideal para contabilizar eventos que ocurren en la aplicacion: peticiones HTTP, errores, transacciones procesadas o usuarios conectados.

```java
@Service
public class PedidoService {

    private final Counter contadorPedidos;
    private final Counter contadorErrores;

    public PedidoService(MeterRegistry registry) {
        this.contadorPedidos = Counter.builder("app.pedidos.creados")
            .description("Numero total de pedidos creados")
            .tag("origen", "api")
            .register(registry);

        this.contadorErrores = Counter.builder("app.pedidos.errores")
            .description("Numero de errores al crear pedidos")
            .register(registry);
    }

    public Pedido crear(PedidoDTO dto) {
        try {
            Pedido pedido = processPedido(dto);
            contadorPedidos.increment();
            return pedido;
        } catch (Exception e) {
            contadorErrores.increment();
            throw e;
        }
    }
}
```

**Timer** mide la duracion de operaciones y la distribucion de tiempos. Es el tipo de metrica mas util para monitorizar latencia de endpoints, duracion de consultas a base de datos o tiempos de respuesta de servicios externos.

```java
@Component
public class ConsultaTimerAspect {

    private final MeterRegistry registry;

    public ConsultaTimerAspect(MeterRegistry registry) {
        this.registry = registry;
    }

    @Around("@annotation(Timed)")
    public Object medirTiempo(ProceedingJoinPoint joinPoint) throws Throwable {
        Timer.Sample sample = Timer.start(registry);
        try {
            Object resultado = joinPoint.proceed();
            sample.stop(Timer.builder("app.consulta.tiempo")
                .tag("metodo", joinPoint.getSignature().getName())
                .register(registry));
            return resultado;
        } catch (Exception e) {
            sample.stop(Timer.builder("app.consulta.tiempo")
                .tag("metodo", joinPoint.getSignature().getName())
                .tag("resultado", "error")
                .register(registry));
            throw e;
        }
    }
}
```

**DistributionSummary** captura la distribucion de valores numericos: tamanos de peticion, numero de elementos en una respuesta o duraciones de procesamiento por lotes.

```java
DistributionSummary.builder("app.pedido.tamano")
    .description("Numero de items por pedido")
    .publishPercentiles(0.5, 0.75, 0.95, 0.99)
    .register(registry)
    .record(pedido.getItems().size());
```

## MeterRegistry

`MeterRegistry` es el registro central donde se almacenan y gestionan todas las metricas. Spring Boot autoconfigura un `CompositeMeterRegistry` que puede exportar a multiples sistemas simultaneamente.

```java
@Configuration
public class MetricsConfig {

    @Bean
    public MeterCustomizer metricasPersonalizadas() {
        return registry -> {
            // Tags globales que se aplican a todas las metricas
            Tags tags = Tags.empty()
                .and("app", "mi-aplicacion")
                .and("version", "1.0.0");

            registry.config().commonTags(tags);
        };
    }
}
```

Los tags son dimensiones que permiten filtrar y agrupar metricas. Por ejemplo, una metrica de peticiones HTTP puede tener tags para `method`, `status` y `uri`, permitiendo consultar solo las peticiones POST que retornaron error 500.

## Exportadores

Micrometer soporta exportacion a multiples sistemas de monitorizacion a traves de dependencias especificas:

```xml
<!-- Prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- Datadog -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-datadog</artifactId>
</dependency>
```

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
        step: 1m
      datadog:
        enabled: true
        api-key: ${DATADOG_API_KEY}
        step: 30s
```

Spring Boot expone automaticamente metricas de la JVM (memoria, GC, threads), del sistema operativo (CPU, disco), de la aplicacion HTTP (peticiones, latencia por endpoint) y del contexto de Spring (beans, uptime). Estas metricas基础icas proporcionan visibilidad inmediata sobre el estado de la aplicacion sin instrumentacion adicional.

> Las metricas de percentiles (p50, p95, p99) son mas informativas que las medias. Un percentil 99 de 500ms indica que el 1% de las peticiones tarda mas de 500ms, lo que revela problemas de latencia que una media de 50ms ocultaria.

