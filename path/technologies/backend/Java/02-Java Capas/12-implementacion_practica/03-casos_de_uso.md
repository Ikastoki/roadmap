# Casos de uso

Cuatro operaciones ilustran el flujo completo a traves de las capas: el CRUD de usuario, la creacion de una orden con validacion de stock, la busqueda mediante especificaciones y la paginacion. Cada caso recorre el mismo camino --controlador, servicio, repositorio-- pero con distintas variantes de logica de negocio.

## CRUD de usuario

El ciclo de vida completo de un usuario abarca creacion, consulta, actualizacion y eliminacion. Cada operacion cruza las tres capas aplicando validaciones especificas.

**Creacion** recibe un `CreateUserRequest`, valida que los campos no esten vacios y que el email no exista previamente, y persiste el nuevo usuario.

```java
// UserService
public UserDTO createUser(CreateUserRequest request) {
    if (userRepository.existsByEmail(request.email())) {
        throw new DuplicateResourceException("El email " + request.email() + " ya esta registrado");
    }
    User user = new User(request.name(), request.email());
    User saved = userRepository.save(user);
    return UserDTO.fromDomain(saved);
}
```

**Lectura** por ID localiza la entidad o lanza `NotFoundException`.

```java
public UserDTO findById(Long id) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("User", id));
    return UserDTO.fromDomain(user);
}
```

**Listado** retorna todos los usuarios sin filtro, util para poblacion inicial de combos o tablas simples.

```java
public List<UserDTO> findAll() {
    return userRepository.findAll().stream()
        .map(UserDTO::fromDomain)
        .toList();
}
```

**Actualizacion** modifica nombre y/o email. Si el email cambia, verifica que el nuevo no este ocupado por otro usuario. El metodo `changeEmail` del modelo de dominio protege el formato del email.

```java
public UserDTO update(Long id, UpdateUserRequest request) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("User", id));

    if (request.name() != null && !request.name().isBlank()) {
        user.setName(request.name());
    }
    if (request.email() != null && !request.email().isBlank()) {
        if (!request.email().equals(user.getEmail())
                && userRepository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("El email " + request.email() + " ya esta registrado");
        }
        user.changeEmail(request.email());
    }

    User saved = userRepository.save(user);
    return UserDTO.fromDomain(saved);
}
```

**Eliminacion** verifica existencia previa y remueve el registro.

```java
public void delete(Long id) {
    if (!userRepository.existsById(id)) {
        throw new NotFoundException("User", id);
    }
    userRepository.deleteById(id);
}
```

## Creacion de orden con validacion de stock

El flujo de creacion de una orden implica multiples repositorios y una regla de negocio critica: no se puede vender mas stock del disponible.

```java
// OrderService
public OrderDTO createOrder(CreateOrderRequest request) {
    // 1. Validar que el usuario existe
    if (!userRepository.existsById(request.userId())) {
        throw new NotFoundException("User", request.userId());
    }

    // 2. Validar stock y preparar items
    List<OrderItem> items = new ArrayList<>();
    for (CreateOrderRequest.ItemDTO itemDTO : request.items()) {
        Product product = productRepository.findById(itemDTO.productId())
            .orElseThrow(() -> new NotFoundException("Product", itemDTO.productId()));

        // La entidad Product lanza BusinessException si no hay stock
        product.reduceStock(itemDTO.quantity());

        items.add(new OrderItem(
            product.getId(),
            product.getName(),
            product.getPrice(),
            itemDTO.quantity()
        ));
    }

    // 3. Crear y persistir la orden
    Order order = new Order(request.userId(), items);
    Order saved = orderRepository.save(order);

    return OrderDTO.fromDomain(saved);
}
```

La validacion de stock reside en la entidad `Product`, no en el servicio. Esto mantiene la regla cerca de los datos que la definen:

```java
// Product.reduceStock
public void reduceStock(int quantity) {
    if (quantity <= 0) {
        throw new IllegalArgumentException("La cantidad debe ser positiva");
    }
    if (quantity > stock) {
        throw new BusinessException(
            "Stock insuficiente para " + name + ": disponible " + stock + ", solicitado " + quantity
        );
    }
    stock -= quantity;
}
```

> La regla "no se puede reducir stock por debajo de cero" esta encapsulada en `Product`, no en `OrderService`. Si en el futuro cambia la politica (por ejemplo, permitir stock negativo para pedidos pendientes de fabricacion), solo se modifica `Product`.

## Busqueda con especificaciones

Cuando los criterios de busqueda son variables y combinables, el patron Specification permite construir predicados de forma composicional sin explosion de metodos en el repositorio.

```java
// business/specification/Specification.java
package com.example.business.specification;

public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) || other.isSatisfiedBy(candidate);
    }

    default Specification<T> not() {
        return candidate -> !this.isSatisfiedBy(candidate);
    }

    static <T> Specification<T> alwaysTrue() {
        return candidate -> true;
    }
}
```

Las especificaciones concretas se definen como metodos estaticos en una clase agrupadora:

```java
// business/specification/UserSpecification.java
package com.example.business.specification;

import com.example.business.model.User;

public class UserSpecification {

    public static Specification<User> byName(String name) {
        return user -> user.getName().toLowerCase().contains(name.toLowerCase());
    }

    public static Specification<User> byEmail(String email) {
        return user -> user.getEmail().equalsIgnoreCase(email);
    }

    public static Specification<User> active() {
        return user -> user.isActive();
    }

    public static Specification<User> createdAfter(LocalDateTime date) {
        return user -> user.getCreatedAt().isAfter(date);
    }
}
```

El repositorio incluye un metodo generico que acepta una especificacion:

```java
// UserRepository
List<User> findBySpecification(Specification<User> spec);

// Implementacion en InMemoryUserRepository
@Override
public List<User> findBySpecification(Specification<User> spec) {
    return store.values().stream()
        .filter(spec::isSatisfiedBy)
        .collect(Collectors.toList());
}
```

Uso desde el servicio:

```java
public List<UserDTO> searchUsers(String name, String email, LocalDateTime createdAfter) {
    Specification<User> spec = Specification.alwaysTrue();

    if (name != null && !name.isBlank()) {
        spec = spec.and(UserSpecification.byName(name));
    }
    if (email != null && !email.isBlank()) {
        spec = spec.and(UserSpecification.byEmail(email));
    }
    if (createdAfter != null) {
        spec = spec.and(UserSpecification.createdAfter(createdAfter));
    }

    return userRepository.findBySpecification(spec).stream()
        .map(UserDTO::fromDomain)
        .toList();
}
```

> El patron Specification evita tener que crear un metodo de repositorio distinto para cada combinacion de filtros. La logica de combinacion permanece en el servicio, que decide que filtros aplicar segun los parametros recibidos.

## Paginacion

Cuando el volumen de datos crece, retornar listas completas es inviable. La paginacion divide los resultados en paginas y proporciona metadatos para la navegacion.

```java
// PaginatedResponse
package com.example.presentation.response;

import java.util.List;

public class PaginatedResponse<T> {
    private final List<T> items;
    private final long totalItems;
    private final int totalPages;
    private final int currentPage;
    private final int pageSize;

    public PaginatedResponse(List<T> items, long totalItems, int page, int pageSize) {
        this.items = items;
        this.totalItems = totalItems;
        this.currentPage = page;
        this.pageSize = pageSize;
        this.totalPages = (int) Math.ceil((double) totalItems / pageSize);
    }

    public List<T> getItems() { return items; }
    public long getTotalItems() { return totalItems; }
    public int getTotalPages() { return totalPages; }
    public int getCurrentPage() { return currentPage; }
    public int getPageSize() { return pageSize; }
    public boolean hasNext() { return currentPage < totalPages; }
    public boolean hasPrevious() { return currentPage > 1; }
}
```

El repositorio acepta pagina y tamano:

```java
// UserRepository
PaginatedResponse<User> findAllPaginated(int page, int pageSize);

// Implementacion en InMemoryUserRepository
@Override
public PaginatedResponse<User> findAllPaginated(int page, int pageSize) {
    List<User> allUsers = List.copyOf(store.values());
    int totalItems = allUsers.size();
    int fromIndex = (page - 1) * pageSize;

    if (fromIndex >= totalItems) {
        return new PaginatedResponse<>(List.of(), totalItems, page, pageSize);
    }

    int toIndex = Math.min(fromIndex + pageSize, totalItems);
    List<User> pageItems = allUsers.subList(fromIndex, toIndex);

    return new PaginatedResponse<>(pageItems, totalItems, page, pageSize);
}
```

El servicio expone el metodo paginado:

```java
public PaginatedResponse<UserDTO> findAllPaginated(int page, int pageSize) {
    PaginatedResponse<User> paginated = userRepository.findAllPaginated(page, pageSize);
    List<UserDTO> dtos = paginated.getItems().stream()
        .map(UserDTO::fromDomain)
        .toList();

    return new PaginatedResponse<>(
        dtos,
        paginated.getTotalItems(),
        paginated.getCurrentPage(),
        paginated.getPageSize()
    );
}
```

El controlador lo expone como endpoint:

```java
public ApiResponse listUsersPaginated(int page, int pageSize) {
    try {
        if (page < 1) page = 1;
        if (pageSize < 1) pageSize = 10;
        if (pageSize > 100) pageSize = 100;

        PaginatedResponse<UserDTO> result = userService.findAllPaginated(page, pageSize);
        return ApiResponse.ok(result);
    } catch (Exception e) {
        return exceptionHandler.handle(e);
    }
}
```

> La paginacion debe limitar el tamano maximo de pagina para evitar que un cliente solicite miles de registros en una sola llamada. En este ejemplo se fija un limite superior de 100 elementos por pagina.
