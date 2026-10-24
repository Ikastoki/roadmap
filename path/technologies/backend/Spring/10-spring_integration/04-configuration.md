# Configuration

La configuracion de Spring Integration en aplicaciones modernas se basa principalmente en el Java DSL, que proporciona una API fluida y expressiva para definir flujos de integracion. Aunque la configuracion XML aun existe en proyectos legados, el enfoque basado en anotaciones y el DSL de Java son los recomendados para nuevos desarrollos.

## Java DSL

El Java DSL de Spring Integration permite definir flujos de integracion de forma declarativa utilizando una API de construccion encadenada. Cada flujo se define como un bean de Spring que encapsula la logica de enrutamiento, transformacion y procesamiento de mensajes.

```java
@Configuration
public class IntegrationConfig {

    @Bean
    public IntegrationFlow orderProcessingFlow() {
        return IntegrationFlow.from("rawOrderChannel")
                .filter(Order.class, order -> order.getAmount() > 0)
                .transform(Order.class, order -> {
                    order.setStatus(Status.VALIDATED);
                    order.setValidatedAt(LocalDateTime.now());
                    return order;
                })
                .<Order, String>route(
                        Order::getType,
                        mapping -> mapping
                                .subFlowMapping("PURCHASE", sf -> sf
                                        .handle(purchaseService, "processPurchase"))
                                .subFlowMapping("RETURN", sf -> sf
                                        .handle(returnService, "processReturn")))
                .get();
    }

    @Bean
    public IntegrationFlow aggregationFlow() {
        return IntegrationFlow.from("responseChannel")
                .aggregate(config -> config
                        .correlationStrategy(msg -> msg.getHeaders().get("correlationId"))
                        .releaseStrategy(group -> group.size() >= 3)
                        .sendPartialResultOnExpiry(true)
                        .outputChannel("aggregatedResultChannel"))
                .get();
    }
}
```

El DSL ofrece una sintaxis intuitiva donde cada metodo de la cadena representa una etapa del flujo. Los metodos como `filter`, `transform`, `route`, `split`, `aggregate` y `handle` mapean directamente a los patrones de integracion.

## @IntegrationComponentScan

La anotacion `@IntegrationComponentScan` habilita el escaneo de componentes de Spring Integration en el classpath. Es necesaria cuando se utilizan anotaciones como `@ServiceActivator`, `@Transformer` o `@Filter` directamente en metodos de beans, en lugar de definir los flujos de forma programatica.

```java
@SpringBootApplication
@IntegrationComponentScan
public class IntegrationApplication {

    public static void main(String[] args) {
        SpringApplication.run(IntegrationApplication.class, args);
    }

    @ServiceActivator(inputChannel = "inputChannel", outputChannel = "outputChannel")
    public String processMessage(String payload) {
        return payload.toUpperCase();
    }

    @Transformer(inputChannel = "rawChannel", outputChannel = "transformedChannel")
    public Order transformOrder(RawData raw) {
        return new Order(raw.getId(), raw.getValue());
    }

    @Filter(inputChannel = "filterChannel", outputChannel = "validChannel")
    public boolean validateOrder(Order order) {
        return order.getAmount() > 0 && order.getCustomerId() != null;
    }
}
```

## MessageChannel

Los canales de mensajes son los conductos que conectan los componentes de integracion. Spring Integration ofrece multiples implementaciones de canales con diferentes semanticas de entrega.

```java
// Canal directo (sincrono, punto a punto)
@Bean
public DirectChannel orderChannel() {
    return new DirectChannel();
}

// Canal de difusion (asincrono, publicacion-suscripcion)
@Bean
public PublishSubscribeChannel notificationChannel() {
    return new PublishSubscribeChannel(taskExecutor());
}

// Canal con queue (asincrono, punto a punto)
@Bean
public QueueChannel asyncProcessingChannel() {
    return new QueueChannel(100);  // capacidad maxima de 100 mensajes
}

// Canal de prioridad
@Bean
public PriorityChannel priorityChannel() {
    return new PriorityChannel(200,
            Comparator.comparingInt(msg -> (int) msg.getHeaders().get("priority")));
}

@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(50);
    return executor;
}
```

> La seleccion del tipo de canal tiene implicaciones significativas en el comportamiento del flujo. Los canales directos ejecutan la logica en el hilo del productor, mientras que los canales con queue delegan la ejecucion al hilo del consumidor. Esto impacta directamente en la latencia y el throughput del sistema.

## MessageHandler

El MessageHandler es la interfaz que implementan todos los componentes que procesan mensajes en Spring Integration. Cada adaptador, transformer y service activator interno utiliza un MessageHandler para ejecutar la logica de procesamiento.

```java
@Bean
public MessageHandler customHandler() {
    return message -> {
        String payload = (String) message.getPayload();
        String correlationId = message.getHeaders().get("correlationId", String.class);

        try {
            Result result = externalService.process(payload);
            message.getHeaders().getChannel().send(
                    MessageBuilder.withPayload(result)
                            .setHeader("correlationId", correlationId)
                            .build());
        } catch (Exception e) {
            messagingExceptionHandler.handleException(message, e);
        }
    };
}

@Bean
@ServiceActivator(inputChannel = "customChannel")
public MessageHandler configuredHandler() {
    return customHandler();
}
```

## IntegrationFlow

El IntegrationFlow es la unidad de construccion principal del DSL. Representa un pipeline completo de procesamiento de mensajes que se construye encadenando operaciones. Un flujo tiene un punto de entrada (que puede ser un canal, un adaptador o un poller) y termina con un handler o un canal de salida.

```java
@Bean
public IntegrationFlow complexFlow(JdbcTemplate jdbcTemplate,
                                   ThreadPoolTaskExecutor executor) {
    return IntegrationFlow.from(
                    Jdbc.inboundChannelAdapter(jdbcTemplate,
                                    "SELECT * FROM events WHERE processed = false")
                            .updateSql("UPDATE events SET processed = true WHERE id = :id")
                            .rowMapper(new BeanPropertyRowMapper<>(Event.class)),
                    e -> e.poller(Pollers.fixedRate(10000).maxMessagesPerPoll(50)))
            .log(LoggingHandler.Level.INFO, "com.example.integration", "events")
            .split()
            .channel(c -> executor)
            .<Event, String>transform(event -> event.getType().name() + ":" + event.getPayload())
            .filter(String.class, payload -> !payload.isBlank())
            .handle(Jdbc.outboundAdapter(jdbcTemplate)
                    .sql("INSERT INTO processed_events (event_data) VALUES (:payload)"))
            .get();
}
```

La modularizacion de los flujos es una practica recomendada: en lugar de definir flujos monoliticos complejos, se decomponen en flujos mas pequenos que se conectan a traves de canales. Esto facilita las pruebas, el mantenimiento y la reutilizacion de componentes.

> Los flujos de integracion pueden anidarse utilizando sub-flows. Esto permite construir arquitecturas jerarquicas donde un flujo principal delega a flujos especializados segun el tipo de mensaje o el resultado de un router.

La combinacion del Java DSL con las anotaciones de Spring Integration ofrece dos enfoques complementarios: el DSL para flujos complejos con logica de enrutamiento y transformacion, y las anotaciones para puntos de integracion simples que se benefician de la declaratividad. En la practica, la mayoria de las aplicaciones combinan ambos enfoques segun la complejidad de cada flujo.

