# Components

Spring Extension estructura la integracion empresarial en torno a un conjunto de componentes bien definidos, cada uno implementando un patron especifico de los Enterprise Integration Patterns. Comprender la responsabilidad de cada componente y como se combinan es fundamental para construir flujos de integracion correctos y mantenibles.

## MessageProducer

El MessageProducer es el componente que genera mensajes y los introduce en un canal. Puede ser un adaptador que recibe datos de un sistema externo (un archivo, una peticion HTTP, un mensaje JMS) o un componente interno que genera mensajes a partir de logica de negocio. Su responsabilidad es encapsular la obtencion de datos y transformarlos en mensajes validos con payload y headers apropiados.

```java
@Bean
public IntegrationFlow fileWatcherFlow() {
    return IntegrationFlow.from(
                    Files.inboundAdapter(new File("/data/inbox"))
                            .patternFilter("*.csv")
                            .preventDuplicates(true),
                    e -> e.poller(Pollers.fixedDelay(5000)))
            .handle(message -> {
                File payload = (File) message.getPayload();
                // procesar archivo
            })
            .get();
}
```

## MessageConsumer

El MessageConsumer es el componente que recibe mensajes de un canal y ejecuta la logica de negocio asociada. Puede procesar el mensaje de forma sincrona o asincrona, y tiene la responsabilidad de gestionar el resultado del procesamiento (confirmacion, rechazo o reencolamiento).

```java
@Bean
public MessageHandler orderHandler() {
    return message -> {
        Order order = (Order) message.getPayload();
        orderService.process(order);
    };
}
```

## Service Activator

El Service Activator conecta un canal de entrada con un bean de servicio que procesa el mensaje. Es el endpoint mas generico de Spring Integration, ya que puede invocar cualquier metodo de cualquier bean como respuesta a la recepcion de un mensaje.

```java
@Bean
public IntegrationFlow serviceActivatorFlow(OrderService orderService) {
    return IntegrationFlow.from("orderInputChannel")
            .handle(orderService, "processOrder")
            .get();
}
```

El Service Activator admite tanto invocaciones sincronas como asincronas, y puede configurarse con un error handler para manejar excepciones lanzadas durante el procesamiento.

## Transformer

El Transformer convierte el payload de un mensaje de un formato a otro. Es uno de los componentes mas utilizados en la integracion empresarial, ya que los sistemas externos raramente comparten los mismos formatos de datos.

```java
@Bean
public IntegrationFlow transformerFlow() {
    return IntegrationFlow.from("rawDataChannel")
            .transform(String.class, payload -> payload.toUpperCase().trim())
            .transform(OrderCsv.class, csv -> {
                Order order = new Order();
                order.setId(csv.getId());
                order.setTotal(new BigDecimal(csv.getTotal()));
                return order;
            })
            .channel("processedDataChannel")
            .get();
}
```

El Transformer puede aplicarse de forma declarativa utilizando expressions SpEL o de forma programatica implementando la interfaz `GenericTransformer`.

## Filter

El Filter evalua si un mensaje debe continuar fluyendo por el canal o debe ser descartado. Retorna un booleano que determina el destino del mensaje: `true` permite el paso, `false` lo bloquea. Los filtros son esenciales para reducir el ruido en los canales y asegurar que solo los mensajes relevantes alcancen su destino.

```java
@Bean
public IntegrationFlow filterFlow() {
    return IntegrationFlow.from("inputChannel")
            .filter(Order.class, order -> order.getTotal().compareTo(BigDecimal.ZERO) > 0)
            .channel("validOrdersChannel")
            .get();
}
```

> Cuando un filtro descarta un mensaje, el framework puede redirigirlo a un canal de dead-letter en lugar de descartarlo silenciosamente. Esto es util para revision posterior de los mensajes rechazados.

## Router

El Router dirige los mensajes a diferentes canales segun su contenido o algun criterio definido. Es el componente que implementa la logica de enrutamiento condicional, permitiendo que un flujo se bifurque en multiples caminos segun el tipo, el valor o las caracteristicas del payload.

```java
@Bean
public IntegrationFlow routerFlow() {
    return IntegrationFlow.from("incomingChannel")
            .<Order, String>route(
                    order -> order.getType().name(),
                    mapping -> mapping
                            .subFlowMapping("DOMESTIC", sf -> sf
                                    .channel("domesticProcessingChannel"))
                            .subFlowMapping("INTERNATIONAL", sf -> sf
                                    .channel("internationalProcessingChannel"))
                            .defaultSubFlowMapping(sf -> sf
                                    .channel("unknownTypeChannel")))
            .get();
}
```

## Splitter

El Splitter toma un mensaje con un payload que contiene multiples elementos y lo divide en multiples mensajes individuales, cada uno con un fragmento del payload original. Esto es fundamental cuando un sistema fuente envia lotes de datos que deben procesarse de forma independiente.

```java
@Bean
public IntegrationFlow splitterFlow() {
    return IntegrationFlow.from("batchChannel")
            .split(BatchOrder.class, BatchOrder::getItems)
            .channel("individualItemChannel")
            .get();
}
```

## Aggregator

El Aggregator es el componente opuesto al Splitter. Recibe multiples mensajes individuales y los combina en un unico mensaje compuesto. Requiere una estrategia de correlacion para determinar que mensajes pertenecen al mismo grupo y una estrategia de release para decidir cuando el grupo esta completo.

```java
@Bean
public IntegrationFlow aggregatorFlow() {
    return IntegrationFlow.from("individualResponseChannel")
            .aggregate(config -> config
                    .correlationStrategy(msg -> msg.getHeaders().get("batchId"))
                    .releaseStrategy(group -> group.size() == 10)
                    .outputChannel("aggregatedChannel"))
            .get();
}
```

## Gateway

El Gateway es un interfaz que simplifica la invocacion de flujos de integracion desde la logica de negocio. Proporciona un contrato tipado que oculta los detalles de la infraestructura de mensajeria, permitiendo que los servicios empresariales envien y reciban mensajes sin conocer los canales subyacentes.

```java
@MessagingGateway(defaultRequestChannel = "orderInputChannel",
                  defaultReplyChannel = "orderOutputChannel")
public interface OrderGateway {

    @Gateway(requestTimeout = 5000)
    OrderConfirmation sendOrder(Order order);

    @Gateway(requestTimeout = 10000)
    List<OrderConfirmation> sendBatchOrders(List<Order> orders);
}

// Uso desde un servicio
@Service
public class OrderService {

    private final OrderGateway orderGateway;

    public OrderService(OrderGateway orderGateway) {
        this.orderGateway = orderGateway;
    }

    public OrderConfirmation submitOrder(Order order) {
        return orderGateway.sendOrder(order);
    }
}
```

> Los Gateways pueden definirse con multiples metodos, cada uno mapeado a un flujo de integracion distinto. Esto permite construir fachadas que orquestan multiples flujos de integracion a traves de una interfaz unica.

La composicion de estos componentes permite construir desde flujos simples de punto a punto hasta arquitecturas de integracion complejas con multiples ramas, transformaciones y puntos de agregacion. La clave esta en identificar el patron correcto para cada etapa del flujo y conectar los componentes a traves de canales tipados.

