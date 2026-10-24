# Service

Un Service es la pieza central de la capa de negocio. Es la clase que implementa la logica de la aplicacion, orquesta las operaciones entre repositories, maneja transacciones y aplica las reglas de negocio. Su caracteristica fundamental es que no depende ni de la capa de presentacion ni de la base de datos; trabaja exclusivamente con abstracciones del dominio y repositorios, lo que lo hace independiente de la infraestructura y facilmente testeable.

## Responsabilidades del Service

La responsabilidad principal del Service es contener la logica de negocio de la aplicacion. Esto incluye implementar las reglas que definen como se crean, modifican y eliminan los objetos de dominio, validar que esas reglas se cumplan antes de persistir cualquier cambio, orquestar multiples repositorios cuando una operacion involucra varias entidades, manejar excepciones de negocio y coordinar transacciones para garantizar la atomicidad de las operaciones.

```java
public class TransferService {
    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;
    private final NotificationService notificationService;

    public TransferService(AccountRepository accountRepository,
                           TransactionRepository transactionRepository,
                           NotificationService notificationService) {
        this.accountRepository = accountRepository;
        this.transactionRepository = transactionRepository;
        this.notificationService = notificationService;
    }

    public TransferResult transfer(String fromIban, String toIban, Money amount) {
        Account origin = accountRepository.findByIban(fromIban)
            .orElseThrow(() -> new AccountNotFoundException(fromIban));
        Account destination = accountRepository.findByIban(toIban)
            .orElseThrow(() -> new AccountNotFoundException(toIban));

        origin.withdraw(amount);
        destination.deposit(amount);

        Transaction transaction = new Transaction(origin, destination, amount);
        transactionRepository.save(transaction);

        notificationService.notifyTransfer(origin.getOwner(), transaction);

        return new TransferResult(transaction.getId(), "COMPLETED");
    }
}
```

Un Service no debe contener logica de presentacion (como formatear respuestas HTTP) ni logica de persistencia (como construir queries SQL). Su unica preocupacion es el que del negocio, no el como tecnico.

## Transacciones

El Service es el responsable de definir los limites transaccionales. Una operacion de negocio que involucra multiples escrituras debe ejecutarse dentro de una misma transaccion para garantizar que, si algo falla, todos los cambios se reviertan.

```java
public class OrderService {
    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;
    private final PaymentGateway paymentGateway;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request.userId());

        for (ItemRequest item : request.items()) {
            Product product = inventoryRepository.findById(item.productId())
                .orElseThrow(() -> new ProductNotFoundException(item.productId()));

            order.addItem(product, item.quantity());
            inventoryRepository.decreaseStock(product.getId(), item.quantity());
        }

        PaymentResult payment = paymentGateway.charge(order.getTotal());
        order.markAsPaid(payment.transactionId());

        return orderRepository.save(order);
    }
}
```

En este ejemplo, si el pago falla o el inventario no tiene suficiente stock, toda la operacion se revierte y la base de datos queda en un estado consistente.

## Singleton e inyeccion de dependencias

Los Services son tipicamente objetos sin estado (stateless): no almacenan informacion entre llamadas. Esto permite que una unica instancia del Service sea compartida por toda la aplicacion, aplicando el patron Singleton. En lugar de crear un nuevo Service cada vez que se necesita, se crea una unica instancia y se reutiliza.

En Java puro, el Singleton se implementa con una instancia estatica o un contenedor de dependencias:

```java
public class ServiceLocator {
    private static final UserService userService = new UserService(
        new JdbcUserRepository(),
        new EmailService()
    );

    public static UserService getUserService() {
        return userService;
    }
}
```

En frameworks como Spring, el contenedor de inversion de control gestiona automaticamente el ciclo de vida de los Services. La anotacion `@Service` registra la clase como un bean, y Spring se encarga de crear una unica instancia (por defecto) e inyectarla donde sea necesaria. La inyeccion por constructor es la forma recomendada porque hace explicitas las dependencias, facilita el testing con mocks y evita que el Service quede en un estado inconsistente.

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public UserService(UserRepository userRepository,
                       PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
}
```

La inyeccion de dependencias (DI) es el mecanismo que permite que un Service reciba sus dependencias desde fuera en lugar de crearlas internamente. Esto desacopla el Service de sus implementaciones concretas: el Service solo conoce interfaces, no sabe si el repositorio usa JDBC, JPA o una base de datos en memoria. Esto hace que el Service sea testeable sin infraestructura real, simplemente inyectando implementaciones mock durante los tests.

> La regla de oro: un Service nunca debe instanciar sus dependencias con `new`. Si lo hace, esta acoplandose a una implementacion concreta y perdiendo los beneficios del desacoplamiento.

## Service vs Use Case

En arquitecturas como Clean Architecture o Hexagonal, el termino Service se refina en dos conceptos: los Use Cases (o Interactors) representan una operacion atomica del negocio, mientras que los Services agrupan varios casos de uso relacionados. Un Use Case suele tener un unico metodo publico (`execute`), mientras que un Service agrupa varios metodos que operan sobre el mismo dominio.

```java
// Use Case: unica operacion atomica
public class CreateUserUseCase {
    private final UserRepository userRepository;

    public User execute(CreateUserRequest request) {
        // logica especifica para crear un usuario
    }
}

// Service: agrupacion de operaciones relacionadas
public class UserService {
    private final CreateUserUseCase createUser;
    private final UpdateUserUseCase updateUser;
    private final DeleteUserUseCase deleteUser;

    public UserDTO createUser(CreateUserRequest request) {
        return createUser.execute(request);
    }
}
```

Ambos enfoques son validos. La eleccion depende de la complejidad de la aplicacion: para aplicaciones pequenas, un Service con varios metodos es suficiente; para aplicaciones grandes con muchos casos de uso, separar cada operacion en su propia clase mejora la organizacion y la adherencia al principio de responsabilidad unica.
