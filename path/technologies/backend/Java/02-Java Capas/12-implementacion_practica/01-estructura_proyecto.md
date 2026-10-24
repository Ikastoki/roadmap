# Estructura del proyecto

Una aplicacion Java con arquitectura de capas no se distingue por las librerias que emplea, sino por la organizacion de sus elementos. La disposicion de paquetes, la nomenclatura de clases y las dependencias entre modulos constituyen el esqueleto que sostiene la separacion de responsabilidades. Sin una estructura clara, el patron se diluye y el codigo termina degenerando en el mismo acoplamiento que pretendia evitar.

## Organizacion de paquetes

La convencion mas extendida organiza el codigo por capa tecnica, agrupando en un mismo paquete todos los elementos que pertenecen a la misma responsabilidad arquitectonica. Cada capa se convierte asi en un directorio dentro del arbol fuente, con subpaquetes que refinan el proposito de cada clase.

```
src/main/java/com/example/
├── Main.java
├── presentation/
│   ├── controller/
│   │   ├── UserController.java
│   │   └── OrderController.java
│   └── dto/
│       ├── request/
│       │   ├── CreateUserRequest.java
│       │   ├── UpdateUserRequest.java
│       │   └── CreateOrderRequest.java
│       └── response/
│           ├── UserResponse.java
│           ├── OrderResponse.java
│           └── PaginatedResponse.java
├── business/
│   ├── service/
│   │   ├── UserService.java
│   │   └── OrderService.java
│   ├── model/
│   │   ├── User.java
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   └── Product.java
│   └── exception/
│       ├── BusinessException.java
│       ├── NotFoundException.java
│       └── DuplicateResourceException.java
├── data/
│   ├── repository/
│   │   ├── GenericRepository.java
│   │   ├── UserRepository.java
│   │   ├── OrderRepository.java
│   │   ├── ProductRepository.java
│   │   ├── InMemoryUserRepository.java
│   │   ├── InMemoryOrderRepository.java
│   │   └── InMemoryProductRepository.java
│   └── entity/
│       ├── UserEntity.java
│       ├── OrderEntity.java
│       ├── OrderItemEntity.java
│       └── ProductEntity.java
└── config/
    └── AppConfig.java
```

La capa `presentation` contiene exclusivamente clases que gestionan la interaccion con el exterior: controladores que reciben peticiones y las traducen a llamadas de servicio, y DTOs que definen el contrato publico de la API. La capa `business` alberga el corazon de la aplicacion: los modelos de dominio con su comportamiento, los servicios que orquestan operaciones y las excepciones que representan violaciones de las reglas de negocio. La capa `data` abstrae la persistencia mediante repositorios y entidades que reflejan el esquema de la base de datos. El paquete `config` centraliza la creacion de dependencias y la configuracion global.

## Entidades de dominio

El modelo de dominio captura los conceptos del negocio con su estado y su comportamiento. No son meros contenedores de datos, sino objetos que encapsulan reglas e invariantes.

```java
public class User {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }

    public void changeEmail(String newEmail) {
        if (newEmail == null || newEmail.isBlank()) {
            throw new IllegalArgumentException("El email no puede estar vacio");
        }
        this.email = newEmail;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

```java
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
    private int stock;

    public Product(String name, BigDecimal price, int stock) {
        this.name = name;
        this.price = price;
        this.stock = stock;
    }

    public void reduceStock(int quantity) {
        if (quantity > stock) {
            throw new BusinessException("Stock insuficiente para " + name);
        }
        stock -= quantity;
    }

    public void increaseStock(int quantity) {
        stock += quantity;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public int getStock() { return stock; }
}
```

```java
public class Order {
    private Long id;
    private Long userId;
    private List<OrderItem> items;
    private BigDecimal total;
    private LocalDateTime createdAt;

    public Order(Long userId, List<OrderItem> items) {
        this.userId = userId;
        this.items = items;
        this.total = calculateTotal(items);
        this.createdAt = LocalDateTime.now();
    }

    private BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getUserId() { return userId; }
    public List<OrderItem> getItems() { return items; }
    public BigDecimal getTotal() { return total; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

```java
public class OrderItem {
    private Long productId;
    private String productName;
    private BigDecimal price;
    private int quantity;

    public OrderItem(Long productId, String productName, BigDecimal price, int quantity) {
        this.productId = productId;
        this.productName = productName;
        this.price = price;
        this.quantity = quantity;
    }

    public Long getProductId() { return productId; }
    public String getProductName() { return productName; }
    public BigDecimal getPrice() { return price; }
    public int getQuantity() { return quantity; }
}
```

## Repositorios

Los repositorios definen contratos de acceso a datos. La capa de negocio depende de estas interfaces, no de implementaciones concretas.

```java
public interface GenericRepository<T, ID> {
    T save(T entity);
    Optional<T> findById(ID id);
    List<T> findAll();
    void deleteById(ID id);
    boolean existsById(ID id);
    long count();
}
```

```java
public interface UserRepository extends GenericRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
    List<User> findByNameContaining(String name);
    List<User> findAllActive();
}
```

```java
public interface OrderRepository extends GenericRepository<Order, Long> {
    List<Order> findByUserId(Long userId);
    List<Order> findByUserIdOrderByCreatedAtDesc(Long userId);
}
```

```java
public interface ProductRepository extends GenericRepository<Product, Long> {
    Optional<Product> findById(Long id);
    List<Product> findByStockGreaterThan(int minStock);
}
```

## Servicios

Los servicios orquestan la logica de negocio. Dependen de repositorios y otros servicios, nunca de controladores o detalles de presentacion.

```java
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserDTO createUser(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("El email " + request.email() + " ya esta registrado");
        }
        User user = new User(request.name(), request.email());
        User saved = userRepository.save(user);
        return UserDTO.fromDomain(saved);
    }

    public UserDTO findById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("User", id));
        return UserDTO.fromDomain(user);
    }
}
```

```java
public class OrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;

    public OrderService(OrderRepository orderRepository, ProductRepository productRepository,
                        UserRepository userRepository) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
        this.userRepository = userRepository;
    }

    public OrderDTO createOrder(CreateOrderRequest request) {
        if (!userRepository.existsById(request.userId())) {
            throw new NotFoundException("User", request.userId());
        }

        List<OrderItem> items = new ArrayList<>();
        for (CreateOrderRequest.ItemDTO itemDTO : request.items()) {
            Product product = productRepository.findById(itemDTO.productId())
                .orElseThrow(() -> new NotFoundException("Product", itemDTO.productId()));
            product.reduceStock(itemDTO.quantity());
            items.add(new OrderItem(product.getId(), product.getName(), product.getPrice(), itemDTO.quantity()));
        }

        Order order = new Order(request.userId(), items);
        Order saved = orderRepository.save(order);
        return OrderDTO.fromDomain(saved);
    }
}
```

## Controladores

Los controladores traducen peticiones externas en llamadas a servicios y convierten los resultados en respuestas estructuradas. No contienen logica de negocio ni acceso directo a datos.

```java
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    public Response createUser(CreateUserRequest request) {
        if (request.name() == null || request.name().isBlank()) {
            return Response.badRequest("El nombre es obligatorio").build();
        }
        try {
            UserDTO user = userService.createUser(request);
            return Response.created(user).build();
        } catch (DuplicateResourceException e) {
            return Response.conflict(e.getMessage()).build();
        }
    }

    public Response getUser(Long id) {
        try {
            UserDTO user = userService.findById(id);
            return Response.ok(user).build();
        } catch (NotFoundException e) {
            return Response.notFound(e.getMessage()).build();
        }
    }
}
```

## DTOs

Los DTOs definen el contrato de la API. Separan el modelo de dominio de la representacion externa, evitando exponer detalles internos de persistencia o estructura.

```java
public record UserDTO(Long id, String name, String email, LocalDateTime createdAt) {
    public static UserDTO fromDomain(User user) {
        return new UserDTO(user.getId(), user.getName(), user.getEmail(), user.getCreatedAt());
    }
}

public record CreateUserRequest(String name, String email) {}

public record UpdateUserRequest(String name, String email) {}

public record UserResponse(Long id, String name, String email, LocalDateTime createdAt) {
    public static UserResponse fromDTO(UserDTO dto) {
        return new UserResponse(dto.id(), dto.name(), dto.email(), dto.createdAt());
    }
}
```

```java
public record OrderDTO(Long id, Long userId, List<OrderItemDTO> items, BigDecimal total, LocalDateTime createdAt) {
    public static OrderDTO fromDomain(Order order) {
        List<OrderItemDTO> itemDTOs = order.getItems().stream()
            .map(OrderItemDTO::fromDomain)
            .toList();
        return new OrderDTO(order.getId(), order.getUserId(), itemDTOs, order.getTotal(), order.getCreatedAt());
    }

    public record OrderItemDTO(Long productId, String productName, BigDecimal price, int quantity) {
        public static OrderItemDTO fromDomain(OrderItem item) {
            return new OrderItemDTO(item.getProductId(), item.getProductName(), item.getPrice(), item.getQuantity());
        }
    }
}

public record CreateOrderRequest(Long userId, List<ItemDTO> items) {
    public record ItemDTO(Long productId, int quantity) {}
}
```

## Excepciones

Las excepciones personalizadas transmiten errores de negocio de forma semantica. Cada tipo de excepcion representa una categoria distinta de fallo, permitiendo al manejador global responder con el codigo HTTP adecuado.

```java
public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}

public class NotFoundException extends RuntimeException {
    private final String resourceName;
    private final Long resourceId;

    public NotFoundException(String resourceName, Long resourceId) {
        super(resourceName + " con id " + resourceId + " no encontrado");
        this.resourceName = resourceName;
        this.resourceId = resourceId;
    }

    public String getResourceName() { return resourceName; }
    public Long getResourceId() { return resourceId; }
}

public class DuplicateResourceException extends BusinessException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

## Clase de configuracion

La configuracion centraliza la creacion de dependencias. En ausencia de un contenedor IoC como Spring, una clase `AppConfig` fabrica y ensambla los objetos del grafo de dependencias.

```java
public class AppConfig {
    private final UserRepository userRepository = new InMemoryUserRepository();
    private final ProductRepository productRepository = new InMemoryProductRepository();
    private final OrderRepository orderRepository = new InMemoryOrderRepository();
    private final UserService userService = new UserService(userRepository);
    private final OrderService orderService = new OrderService(orderRepository, productRepository, userRepository);
    private final UserController userController = new UserController(userService);
    private final OrderController orderController = new OrderController(orderService);
    private final GlobalExceptionHandler exceptionHandler = new GlobalExceptionHandler();

    public UserController userController() { return userController; }
    public OrderController orderController() { return orderController; }
    public GlobalExceptionHandler exceptionHandler() { return exceptionHandler; }
}
```

> La estructura por capa tecnica es la mas habitual en proyectos que usan Spring y Jakarta EE. Una alternativa igualmente valida es la organizacion por dominio (un paquete por cada concepto de negocio, cada uno con sus propias subcapas), que suele preferirse en dominios extensos con equipos grandes.
