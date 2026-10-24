# Clean Architecture

Robert C. Martin (Uncle Bob) publico en 2012 la Clean Architecture como una sintesis de las mejores ideas de la arquitectura hexagonal, la Onion, el Boundary-Control-Entity de Ivar Jacobson y el Screaming Architecture. El resultado es un modelo que combina la rigurosidad estructural de la Onion con el mecanismo de desacoplamiento de puertos y adaptadores, anadiendo una regla de dependencia explicita y una nomenclatura de capas estandarizada que facilita la comunicacion entre equipos.

## La Dependency Rule

La regla fundamental de la Clean Architecture es la Dependency Rule: las dependencias de codigo fuente solo pueden apuntar hacia adentro. Nada en un circulo interior puede saber nada acerca de algo en un circulo exterior. En particular, el nombre de algo declarado en un circulo exterior no debe ser mencionado por el codigo en un circulo interior. Esto incluye clases, funciones, variables o cualquier entidad nombrada.

Consecuencias directas de esta regla: los circulos externos son mecanismos que pueden cambiar sin afectar al nucleo; los circulos internos contienen las reglas de negocio y son independientes de frameworks, bases de datos, interfaces de usuario y dispositivos externos. Un cambio en la base de datos o en el framework web nunca deberia forzar un cambio en las entidades o en los casos de uso.

## Las cuatro capas

**Entities:** las entidades empresariales (enterprise business rules). Encapsulan las reglas de negocio mas generales y de mas alto nivel. Pueden ser objetos con metodos, o simplemente estructuras de datos. Son independientes de la aplicacion concreta: una entidad `Loan` serviria para una aplicacion de prestamos hipotecarios, de coches o personales.

**Use Cases:** las reglas de negocio especificas de la aplicacion (application business rules). Orquestan el flujo de datos hacia y desde las entidades, y dirigen a las entidades para que ejecuten las reglas de negocio. No saben nada sobre la web, la base de datos ni cualquier detalle externo. Un caso de uso `CreateOrderUseCase` sabe que debe validar el stock y calcular el total, pero no sabe si la peticion llego por HTTP, gRPC o una cola de mensajes.

**Interface Adapters:** convierten los datos del formato mas conveniente para los casos de uso y entidades al formato mas conveniente para agentes externos. Aqui viven los controladores REST, los presentadores, los gateways de repositorio y los DTOs. Un controller recibe un JSON, lo convierte en un request object del caso de uso, invoca el caso de uso, recibe el resultado y lo convierte en una respuesta HTTP.

**Frameworks and Drivers:** la capa mas externa, compuesta por frameworks, herramientas, bases de datos, drivers, el servidor web, la IU. Aqui se produce el glue code que conecta el sistema con el mundo real. Todo en esta capa es detalle.

```
┌─────────────────────────────────────┐
│  Frameworks & Drivers               │
│  ┌───────────────────────────────┐  │
│  │  Interface Adapters           │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Use Cases              │  │  │
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │  Entities         │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

## Boundaries entre capas

Cada cruce entre capas debe realizarse a traves de un boundary. Un boundary es un mecanismo que respeta la Dependency Rule: la capa externa llama a una interfaz definida en la capa interna, y la capa externa proporciona la implementacion concreta. Esto es exactamente el patron Puertos y Adaptadores aplicado a cada frontera.

```java
// Entities (capa mas interna)
public class Order {
    private List<OrderItem> items;
    private Money total;

    public void addItem(Product product, int quantity) {
        items.add(new OrderItem(product, quantity));
        recalculateTotal();
    }

    public Money total() {
        return total;
    }

    private void recalculateTotal() {
        this.total = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Use Case
public class CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;

    public CreateOrderUseCase(
            OrderRepository orderRepository,
            ProductRepository productRepository) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
    }

    public CreateOrderResponse execute(CreateOrderRequest request) {
        Order order = new Order();
        for (var item : request.items()) {
            Product product = productRepository.findById(item.productId())
                .orElseThrow(() -> new ProductNotFoundException(item.productId()));
            order.addItem(product, item.quantity());
        }
        Order saved = orderRepository.save(order);
        return new CreateOrderResponse(saved.id(), saved.total());
    }
}

// Interface Adapters (controller y DTO)
@RestController
public class OrderController {
    private final CreateOrderUseCase createOrderUseCase;

    @PostMapping("/orders")
    public ResponseEntity<CreateOrderResponse> createOrder(
            @RequestBody CreateOrderRequest request) {
        CreateOrderResponse response = createOrderUseCase.execute(request);
        return ResponseEntity.status(201).body(response);
    }
}

// Frameworks/Drivers (implementacion JPA del repositorio)
@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager em;

    @Override
    public Order save(Order order) {
        OrderEntity entity = OrderEntity.from(order);
        em.persist(entity);
        return entity.toDomain();
    }
}
```

> El flujo de control es distinto al flujo de dependencia. El control puede viajar de fuera a dentro (el controller llama al use case), pero la dependencia en tiempo de compilacion viaja en sentido contrario (el controller depende del use case, no al reves).

## Diferencia con Hexagonal y Onion

La Clean Architecture se distingue de la hexagonal y la Onion en tres aspectos. Primero, define una nomenclatura de capas precisa (entities, use cases, interface adapters, frameworks) que estandariza la comunicacion. Segundo, explicita la existencia de boundaries en cada frontera entre capas, con sus correspondientes modelos de datos locales. Tercero, la Dependency Rule se formula como un principio autonomo que trasciende la implementacion concreta, aplicable incluso sin puertos ni adaptadores.

En la practica, los tres enfoques convergen. Un equipo puede comenzar con hexagonal, anadir estructura con Onion y formalizar con Clean Architecture. La mayoria de las aplicaciones que se declaran "hexagonales" en realidad implementan una mezcla de los tres.
