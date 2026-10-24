# Messaging

La comunicacion asincrona entre microservicios es fundamental para construir sistemas resilientes y desacoplados. Spring Cloud Stream proporciona una abstraccion sobre los sistemas de mensajeria (RabbitMQ, Apache Kafka, Amazon Kinesis) que permite a los desarrolladores concentrarse en la logica de negocio sin preocuparse por laspecificidades del broker subyacente. El mismo codigo de productor y consumidor funciona con cualquier broker compatible, cambiando unicamente la configuracion.

El concepto central es el **binding**, que conecta los metodos de la aplicacion con canales del broker a traves de una configuracion declarativa. Un productor publica mensajes a un canal bindeado a un destination (exchange o topic) del broker, y un consumidor recibe mensajes de un canal bindeado al mismo destination. La infraestructura de Spring Cloud Stream gestiona la serializacion, el envio, la recepcion y el manejo de errores de forma transparente.

## Spring Cloud Stream

La integracion con Spring Cloud Stream se realiza a traves de dependencias especificas para cada broker. Cada binder implementa la interfaz comun de Spring Cloud Stream, proporcionando la conectividad con el broker y la gestion de canales.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>

<!-- o para RabbitMQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

```java
@SpringBootApplication
public class OrderEventApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderEventApplication.class, args);
    }
}

interface OrderChannels {

    String OUTPUT = "orderOutput";
    String INPUT = "orderInput";

    @Output(OUTPUT)
    MessageChannel orderOutput();

    @Input(INPUT)
    SubscribableChannel orderInput();
}

@Component
@EnableBinding(OrderChannels.class)
public class OrderEventProducer {

    private final OrderChannels channels;

    public OrderEventProducer(OrderChannels channels) {
        this.channels = channels;
    }

    public void publishOrderCreated(Order order) {
        channels.orderOutput().send(
                MessageBuilder.withPayload(order)
                        .setHeader("eventType", "ORDER_CREATED")
                        .build());
    }
}
```

> Desde Spring Cloud Stream 4.x, la anotacion `@EnableBinding` y las interfaces de canales han sido deprecadas en favor del modelo funcional basado en `java.util.function`. El enfoque funcional es mas limpio y se integra mejor con el programacion reactiva.

## Binding

El binding establece la conexion entre los componentes de la aplicacion y los destinos del broker. En el modelo funcional, los beans de tipo `Function`, `Consumer` o `Supplier` se bindean automaticamente a canales del broker segun sus nombres.

```java
@SpringBootApplication
public class OrderStreamApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderStreamApplication.class, args);
    }

    @Bean
    public Function<Order, OrderConfirmation> processOrder() {
        return order -> {
            OrderConfirmation confirmation = orderService.process(order);
            return confirmation;
        };
    }

    @Bean
    public Consumer<OrderEvent> auditOrder() {
        return event -> {
            auditService.log(event);
        };
    }

    @Bean
    public Supplier<HealthStatus> healthCheck() {
        return () -> HealthStatus.ok();
    }
}
```

```yaml
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: orders
          group: order-processor
          content-type: application/json
        processOrder-out-0:
          destination: order-confirmations
          content-type: application/json
        auditOrder-in-0:
          destination: orders
          group: audit-service
          content-type: application/json
      kafka:
        binder:
          brokers: kafka-server:9092
          auto-create-topics: true
```

Los bindings se nombran siguiendo el patron `{functionName}-{in|out}-{index}`, donde `functionName` es el nombre del bean, `in|out` indica la direccion del flujo, e `index` es el numero de canal cuando una funcion tiene multiples entradas o salidas.

## Consumer Groups

Los consumer groups permiten que multiples instancias de un servicio consuman mensajes de un topic sin procesar el mismo mensaje dos veces. Dentro de un group, cada mensaje se entrega a una unica instancia, habilitando el escalado horizontal del consumidor. Cada instancia del grupo recibe un subconjunto disjoint de los mensajes.

```yaml
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: orders
          group: order-processing-group
```

> Sin un consumer group, cada instancia del consumidor recibe todos los mensajes (patron fan-out). Con un group, las instancias comparten la carga (patron competing consumers). La eleccion entre ambos patrones determina si el mensaje se procesa una vez o multiples veces.

## Partitioning

La particionamiento en Spring Cloud Stream distribuye los mensajes entre multiples particiones del topic, donde cada particion es consumida por una unica instancia del consumidor. A diferencia de los consumer groups, que distribuyen mensajes de forma arbitraria, la particionamiento garantiza que mensajes con la misma clave de particion siempre lleguen a la misma instancia.

```yaml
spring:
  cloud:
    stream:
      bindings:
        processOrder-out-0:
          destination: orders
          producer:
            partition-key-expression: headers['orderId']
            partition-count: 4
        processOrder-in-0:
          destination: orders
          consumer:
            partitioned: true
            instance-index: ${INSTANCE_INDEX:0}
            instance-count: 4
```

La particion es especialmente util cuando el orden de procesamiento importa para un subconjunto de mensajes. Los mensajes con el mismo `orderId` se procesan siempre en la misma instancia, preservando la consistencia de estado.

## Error Handling

Spring Cloud Stream ofrece multiples estrategias para manejar errores en la consumicion de mensajes. Un error puede reintentarse, enviarse a un topic de dead-letter, registrarse en un log o descartarse, segun la politica configurada.

```yaml
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: orders
          group: order-processor
          consumer:
            max-attempts: 3
      rabbit:
        binder:
          admin-addresses: rabbitmq-server
          nodes: rabbitmq-server:5672
        bindings:
          processOrder-in-0:
            consumer:
              auto-bind-dlq: true
              dlq-ttl: 86400000
              republish-to-dlq: true
      kafka:
        bindings:
          processOrder-in-0:
            consumer:
              enable-dlq: true
              dlq-name: orders.dlq
              dead-letter-max-attempts: 5
```

Cuando `republish-to-dlq` esta habilitado, el mensaje fallido se reenvia al dead-letter queue con la excepcion original como header, facilitando el diagnosticio y la reprocesamiento posterior.

La combinacion de bindings declarativos, consumer groups y estrategias de error handling permite construir sistemas de mensajeria robustos donde la logica de produccion y consumicion se mantiene limpia y desacoplada de los detalles del broker subyacente.

