# Distributed Tracing

En arquitecturas de microservicios, una sola peticion del usuario puede atravesar multiples servicios antes de completarse. Cuando un error ocurre o un endpoint presenta latencia inaceptable, determinar en que servicio se produce el cuello de botella o la excepcion requiere correlacionar logs分散 across dozens de instancias. La trazabilidad distribuida resuelve este problema asignando identificadores unicos a cada peticion y propagandolos a traves de todos los servicios involucrados.

Spring extension incorpora Micrometer Tracing (anteriormente Spring Cloud Sleuth) como la solucion nativa para la instrumentacion de trazas distribuidas. Micrometer Tracing no implementa el almacenamiento ni la visualizacion de trazas; en su lugar, genera los identificadores de traza y los propaga a traves de las cabeceras HTTP, colas de mensajes y llamadas a base de datos, permitiendo que herramientas externas como Zipkin o Jaeger recopilen y visualicen los datos.

## Trace ID y Span ID

Cada peticion distribuida se identifica con dos valores fundamentales: el **Trace ID**, un identificador unico que se mantiene constante a lo largo de toda la vida de la peticion, y el **Span ID**, que identifica cada operacion individual dentro de esa peticion. Un Trace ID agrupa multiples Spans que representan las llamadas a cada servicio, cada consulta a base de datos y cada operacion de mensajeria.

```
[Trace ID: abc123]
    |
    +-- [Span: order-service (abc123-def456)]
    |       |
    |       +-- [Span: inventory-service (abc123-ghi789)]
    |       |       |
    |       |       +-- [Span: DB query (abc123-jkl012)]
    |       |
    |       +-- [Span: payment-service (abc123-mno345)]
    |
    +-- [Span: notification-service (abc123-pqr678)]
```

## Micrometer Tracing

La integracion de Micrometer Tracing con Spring Boot se realiza a traves de dependencias que habilitan la instrumentacion automatica de llamadas HTTP, clientes de servicios, messaging y acceso a datos.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
    propagation:
      type: b3
  zipkin:
    tracing:
      endpoint: http://zipkin-server:9411/api/v2/spans
```

La propiedad `management.tracing.sampling.probability` controla que proporcion de las peticiones se instrumentan. En produccion, establecer este valor en 1.0 (100%) puede generar un volumen significativo de datos. Un valor de 0.1 (10%) es razonable para sistemas de alto trafico, manteniendo suficiente visibilidad sin impactar el rendimiento.

> Las cabeceras de traza se propagan automaticamente a traves de RestTemplate, WebClient, Feign clients, RabbitTemplate y KafkaTemplate. Las operaciones que no utilizan estos componentes (llamadas HTTP raw, sockets directos) requieren instrumentacion manual.

## Zipkin

Zipkin es un sistema de recopilacion y visualizacion de trazas distribuidas originalmente desarrollado por Twitter. Recopila los Spans generados por los servicios a traves de un endpoint HTTP y los almacena para su consulta y visualizacion. Zipkin proporciona una interfaz web que permite buscar trazas por Trace ID, servicio, duracion y fecha.

```yaml
# Configuracion del servidor Zipkin con Docker
# docker-compose.yml
services:
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=http://elasticsearch:9200
```

Los servicios envian sus Spans a Zipkin utilizando el reporter configurado en `management.zipkin.tracing.endpoint`. Zipkin procesa los Spans, los agrupa por Trace ID y expone la informacion a traves de su API e interfaz web.

## Jaeger

Jaeger es una alternativa open source a Zipkin, desarrollada por Uber y graduada como proyecto de la CNCF. Ofrece funcionalidades adicionales como analisis de dependencias, monitoreo de latencia percentil y deteccion de anomalias. Jaeger soporta el formato de trazas OpenTracing y OpenTelemetry.

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.5
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces
```

> Jaeger soporta tanto el protocolo OTLP (OpenTelemetry Protocol) como el formato Zipkin, lo que permite migrar de un sistema a otro sin modificar la instrumentacion de los servicios.

## Correlacion de Logs

La trazabilidad distribuida alcanza su maximo valor cuando se correlaciona con los logs de la aplicacion. Micrometer Tracing inyecta automaticamente el Trace ID y Span ID en los logs, permitiendo filtrar y buscar logs de todos los servicios involucrados en una peticion especifica.

```yaml
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

```
2024-01-15 10:30:45 INFO  [order-service,abc123def456,789xyz] - Orden recibida
2024-01-15 10:30:45 DEBUG [order-service,abc123def456,789xyz] - Consultando inventario
2024-01-15 10:30:46 INFO  [inventory-service,abc123def456,abc123] - Stock verificado
2024-01-15 10:30:46 INFO  [order-service,abc123def456,789xyz] - Inventario confirmado
2024-01-15 10:30:47 INFO  [payment-service,abc123def456,pqr456] - Pago procesado
```

Con el Trace ID `abc123def456` se pueden filtrar los logs de order-service, inventory-service y payment-service para reconstruir el ciclo completo de la peticion.

## Propagacion a Mensajeria

La propagacion de trazas a traves de sistemas de mensajeria como RabbitMQ o Kafka requiere que los productores inyecten las cabeceras de traza en los mensajes y que los consumidores las extraigan al recibirlos. Spring Extension lo gestiona automaticamente a traves de los interceptores de mensajeria.

```java
// La traza se propaga automaticamente
@Bean
public IntegrationFlow rabbitOutboundFlow(RabbitTemplate rabbitTemplate) {
    return IntegrationFlow.from("orderChannel")
            .handle(Rabbit.outboundAdapter(rabbitTemplate)
                    .exchangeName("orders.exchange"))
            .get();
}
```

La trazabilidad distribuida es un componente esencial para operar microservicios en produccion. Sin ella, la diagnosticacion de problemas se convierte en un proceso manual y tedioso que depende de la correlacion manual de timestamps y mensajes de error分散 en multiples servicios.

