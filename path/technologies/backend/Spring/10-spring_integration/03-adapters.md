# Adapters

Los adaptadores son los componentes que conectan Spring Integration con sistemas externos. Cada adaptador encapsula la complejidad de interactuar con un protocolo o tecnologia especifica, transformando los datos del sistema externo en mensajes de Spring Integration y viceversa. El ecosistema de Spring Integration incluye adaptadores para practicamente todas las tecnologias de integracion empresarial.

## File Adapter

El adaptador de archivos permite leer y escribir datos en el sistema de archivos local o remoto. Soporta operaciones de entrada y salida con multiples formatos (delimitados, posicionales, XML, JSON) y puede configurarse con filtros, expresiones de nombre de archivo y estrategias de polling para detectar nuevos archivos.

```java
@Bean
public IntegrationFlow fileInboundFlow() {
    return IntegrationFlow.from(
                    Files.inboundAdapter(new File("/data/inbox"))
                            .patternFilter("*.csv")
                            .preventDuplicates(true)
                            .autoCreateDirectory(true),
                    e -> e.poller(Pollers.fixedDelay(3000)))
            .transform(Files.toStringTransformer("UTF-8"))
            .channel("fileContentChannel")
            .get();
}

@Bean
public IntegrationFlow fileOutboundFlow() {
    return IntegrationFlow.from("processedChannel")
            .handle(Files.outboundAdapter(new File("/data/outbox"))
                    .fileNameGenerator(msg -> "output_" + System.currentTimeMillis() + ".csv")
                    .autoCreateDirectory(true))
            .get();
}
```

## HTTP Adapter

El adaptador HTTP expone endpoints REST que reciben peticiones y las convierten en mensajes, o envia peticiones HTTP como respuesta a la recepcion de un mensaje. Funciona tanto como inbound (recibe peticiones) como outbound (envia peticiones).

```java
@Bean
public IntegrationFlow httpInboundFlow() {
    return IntegrationFlow.from(
                    Http.inboundGateway("/api/orders")
                            .requestMapping(mapping -> mapping
                                    .methods(HttpMethod.POST)
                                    .consumes("application/json"))
                            .requestPayloadType(Order.class)
                            .replyTimeout(5000))
            .channel("orderInputChannel")
            .get();
}

@Bean
public IntegrationFlow httpOutboundFlow() {
    return IntegrationFlow.from("externalServiceChannel")
            .handle(Http.outboundGateway("https://external-api.com/orders")
                    .httpMethod(HttpMethod.POST)
                    .expectedResponseType(OrderConfirmation.class))
            .get();
}
```

## JMS Adapter

El adaptador JMS permite interactuar con sistemas de mensajeria Java Message Service. Puede escuchar colas y topics, enviar mensajes a destinos JMS, y gestionar transacciones JMS en coordinacion con el transaction manager de la aplicacion.

```java
@Bean
public IntegrationFlow jmsInboundFlow(ConnectionFactory connectionFactory) {
    return IntegrationFlow.from(
                    Jms.messageDrivenChannelAdapter(connectionFactory)
                            .destination("order.queue")
                            .errorChannel("jmsErrorChannel"))
            .transform(Transformers.fromJson(Order.class))
            .channel("orderProcessingChannel")
            .get();
}

@Bean
public IntegrationFlow jmsOutboundFlow(ConnectionFactory connectionFactory) {
    return IntegrationFlow.from("notificationChannel")
            .transform(Transformers.toJson())
            .handle(Jms.outboundAdapter(connectionFactory)
                    .destination("notification.queue"))
            .get();
}
```

> La coordinacion transaccional entre JMS y la base de datos de la aplicacion requiere un `JtaTransactionManager` o un `ChainedTransactionManager` que sincronice las transacciones de ambos recursos.

## AMQP Adapter

El adaptador AMQP conecta Spring Integration con RabbitMQ u otros brokers AMQP. Ofrece una alternativa moderna a JMS con soporte nativo para encolamiento flexible, exchanges y routing keys.

```java
@Bean
public IntegrationFlow amqpInboundFlow(ConnectionFactory connectionFactory) {
    return IntegrationFlow.from(
                    Amqp.inboundAdapter(connectionFactory, "task.queue")
                            .outputChannel("taskChannel")
                            .errorChannel("amqpErrorChannel"))
            .get();
}

@Bean
public IntegrationFlow amqpOutboundFlow(ConnectionFactory connectionFactory) {
    return IntegrationFlow.from("resultChannel")
            .handle(Amqp.outboundAdapter(connectionFactory)
                    .exchangeName("results.exchange")
                    .routingKey("results.routing"))
            .get();
}
```

## Kafka Adapter

El adaptador Kafka permite integrar Spring Integration con Apache Kafka, consumiendo y produciendo mensajes en topics. Soporta configuracion de consumer groups, particiones, offsets y serializadores personalizados.

```java
@Bean
public IntegrationFlow kafkaInboundFlow(ConsumerFactory<String, String> consumerFactory) {
    return IntegrationFlow.from(
                    Kafka.messageDrivenChannelAdapter(consumerFactory,
                                    KafkaMessageDrivenChannelAdapter.ListenerMode.record,
                                    "events.topic")
                            .configureListenerContainer(options -> options
                                    .groupId("integration-group")
                                    .ackMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE)))
            .transform(Transformers.fromJson(Event.class))
            .channel("eventProcessingChannel")
            .get();
}

@Bean
public IntegrationFlow kafkaOutboundFlow(KafkaTemplate<String, String> kafkaTemplate) {
    return IntegrationFlow.from("outboundEventChannel")
            .transform(Transformers.toJson())
            .handle(Kafka.outboundChannelAdapter(kafkaTemplate)
                    .topic("processed-events.topic")
                    .messageKey(msg -> msg.getHeaders().get("eventId", String.class)))
            .get();
}
```

## JDBC Adapter

El adaptador JDBC permite interactuar con bases de datos relacionales a traves de operaciones SQL. Puede ejecutar consultas que generan mensajes, o recibir mensajes que desencadenan la ejecucion de sentencias SQL.

```java
@Bean
public IntegrationFlow jdbcPollingFlow(DataSource dataSource) {
    return IntegrationFlow.from(
                    Jdbc.inboundChannelAdapter(dataSource,
                                    "SELECT * FROM orders WHERE status = 'PENDING'")
                            .updateSql("UPDATE orders SET status = 'PROCESSING' WHERE id IN (:id)")
                            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
                            .maxRows(100),
                    e -> e.poller(Pollers.fixedDelay(5000)))
            .split()
            .channel("orderProcessingChannel")
            .get();
}

@Bean
public IntegrationFlow jdbcOutboundFlow(DataSource dataSource) {
    return IntegrationFlow.from("completedOrdersChannel")
            .handle(Jdbc.outboundAdapter(dataSource)
                    .sql("INSERT INTO order_log (order_id, status, processed_at) VALUES (:id, :status, CURRENT_TIMESTAMP)"))
            .get();
}
```

## Mail Adapter

El adaptador de correo permite recibir y enviar correos electronicos como parte de un flujo de integracion. El adaptador inbound puede conectarse a un servidor IMAP o POP3 para consumir mensajes, mientras que el adaptador outbound envia correos a traves de un servidor SMTP.

```java
@Bean
public IntegrationFlow mailInboundFlow() {
    return IntegrationFlow.from(
                    Mail.imapInboundAdapter(
                                    "imaps://user:password@imap.server.com:993/INBOX")
                            .simpleContent(true)
                            .shouldDeleteMessages(false)
                            .shouldMarkMessagesAsRead(true),
                    e -> e.poller(Pollers.fixedDelay(60000)))
            .transform(Transformers.fromMail())
            .channel("emailProcessingChannel")
            .get();
}

@Bean
public IntegrationFlow mailOutboundFlow(JavaMailSender mailSender) {
    return IntegrationFlow.from("emailOutputChannel")
            .handle(Mail.outboundAdapter(mailSender)
                    .host("smtp.server.com")
                    .port(587)
                    .protocol("smtp"))
            .get();
}
```

> Los adaptadores de mail operan de forma asincrona por naturaleza. El polling sobre buzones IMAP puede generar latencia significativa, por lo que es recomendable ajustar el intervalo de polling segun los requisitos de tiempo real de la aplicacion.

La seleccion del adaptador adecuado depende del sistema externo con el que se desea integrar. Spring Integration permite combinar multiples adaptadores en un mismo flujo, conectando por ejemplo un archivo CSV con una base de datos y un sistema de mensajeria en un unico pipeline de procesamiento.

