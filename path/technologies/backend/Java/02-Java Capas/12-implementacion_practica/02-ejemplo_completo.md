# Ejemplo completo

Un ejemplo integrado muestra como las piezas descritas en la estructura encajan en un flujo real. Desde la definicion del modelo de dominio con comportamiento, pasando por un repositorio generico, hasta el servicio con logica de negocio, el controlador con validacion y el manejador global de errores. Todo ello conectado a traves del configurador de la aplicacion.

## Modelo de dominio con comportamiento

El dominio no es un simple contenedor de datos. `User` encapsula invariantes: el email no puede ser nulo ni vacio, y el metodo `changeEmail` protege esa invariante directamente en la entidad.

```java
package com.example.business.model;

import java.time.LocalDateTime;

public class User {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;

    public User(String name, String email) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("El nombre no puede estar vacio");
        }
        if (email == null || email.isBlank()) {
            throw new IllegalArgumentException("El email no puede estar vacio");
        }
        this.name = name;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }

    public void changeEmail(String newEmail) {
        if (newEmail == null || newEmail.isBlank()) {
            throw new IllegalArgumentException("El email no puede estar vacio");
        }
        if (!newEmail.contains("@")) {
            throw new IllegalArgumentException("El email debe contener @");
        }
        this.email = newEmail;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

```java
package com.example.business.model;

import java.math.BigDecimal;
import java.util.List;

public class Order {
    private Long id;
    private Long userId;
    private List<OrderItem> items;
    private BigDecimal total;
    private LocalDateTime createdAt;

    public Order(Long userId, List<OrderItem> items) {
        if (userId == null) {
            throw new IllegalArgumentException("El usuario es obligatorio");
        }
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("La orden debe tener al menos un item");
        }
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

## Repositorio generico

La interfaz generica define el contrato basico de persistencia. Cada repositorio especifico extiende esta interfaz y anade metodos de busqueda particulares.

```java
package com.example.data.repository;

import java.util.List;
import java.util.Optional;

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
package com.example.data.repository;

import com.example.business.model.User;
import java.util.Optional;

public interface UserRepository extends GenericRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
    List<User> findByNameContaining(String name);
}
```

La implementacion en memoria utiliza un `ConcurrentHashMap` y un contador atomico para los IDs. Es suficiente para desarrollo y pruebas, y puede sustituirse por una implementacion con JDBC o JPA sin modificar el servicio.

```java
package com.example.data.repository;

import com.example.business.model.User;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

public class InMemoryUserRepository implements UserRepository {
    private final Map<Long, User> store = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    @Override
    public User save(User user) {
        if (user.getId() == null) {
            user.setId(idGenerator.getAndIncrement());
        }
        store.put(user.getId(), user);
        return user;
    }

    @Override
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<User> findAll() {
        return List.copyOf(store.values());
    }

    @Override
    public void deleteById(Long id) {
        store.remove(id);
    }

    @Override
    public boolean existsById(Long id) {
        return store.containsKey(id);
    }

    @Override
    public long count() {
        return store.size();
    }

    @Override
    public Optional<User> findByEmail(String email) {
        return store.values().stream()
            .filter(user -> user.getEmail().equals(email))
            .findFirst();
    }

    @Override
    public boolean existsByEmail(String email) {
        return store.values().stream()
            .anyMatch(user -> user.getEmail().equals(email));
    }

    @Override
    public List<User> findByNameContaining(String name) {
        return store.values().stream()
            .filter(user -> user.getName().toLowerCase().contains(name.toLowerCase()))
            .collect(Collectors.toList());
    }
}
```

## Servicio con logica de negocio

El servicio orquesta las operaciones aplicando las reglas de negocio. En el caso de creacion de usuario, valida que el email sea unico antes de persistir.

```java
package com.example.business.service;

import com.example.business.exception.DuplicateResourceException;
import com.example.business.exception.NotFoundException;
import com.example.business.model.User;
import com.example.data.repository.UserRepository;
import com.example.presentation.dto.UserDTO;
import com.example.presentation.dto.request.CreateUserRequest;
import com.example.presentation.dto.request.UpdateUserRequest;

public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserDTO createUser(CreateUserRequest request) {
        if (request.name() == null || request.name().isBlank()) {
            throw new IllegalArgumentException("El nombre es obligatorio");
        }
        if (request.email() == null || request.email().isBlank()) {
            throw new IllegalArgumentException("El email es obligatorio");
        }
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

    public List<UserDTO> findAll() {
        return userRepository.findAll().stream()
            .map(UserDTO::fromDomain)
            .toList();
    }

    public UserDTO update(Long id, UpdateUserRequest request) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("User", id));

        if (request.name() != null && !request.name().isBlank()) {
            user.setName(request.name());
        }
        if (request.email() != null && !request.email().isBlank()) {
            if (!request.email().equals(user.getEmail()) && userRepository.existsByEmail(request.email())) {
                throw new DuplicateResourceException("El email " + request.email() + " ya esta registrado");
            }
            user.changeEmail(request.email());
        }

        User saved = userRepository.save(user);
        return UserDTO.fromDomain(saved);
    }

    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new NotFoundException("User", id);
        }
        userRepository.deleteById(id);
    }
}
```

## Controlador con validacion

El controlador valida la entrada basica antes de delegar al servicio, y traduce las excepciones a respuestas con codigos HTTP semanticos.

```java
package com.example.presentation.controller;

import com.example.business.exception.BusinessException;
import com.example.business.exception.DuplicateResourceException;
import com.example.business.exception.NotFoundException;
import com.example.business.service.UserService;
import com.example.presentation.dto.UserDTO;
import com.example.presentation.dto.request.CreateUserRequest;
import com.example.presentation.dto.request.UpdateUserRequest;
import com.example.presentation.response.ApiResponse;
import com.example.presentation.response.PaginatedResponse;
import java.util.List;

public class UserController {
    private final UserService userService;
    private final GlobalExceptionHandler exceptionHandler;

    public UserController(UserService userService, GlobalExceptionHandler exceptionHandler) {
        this.userService = userService;
        this.exceptionHandler = exceptionHandler;
    }

    public ApiResponse createUser(CreateUserRequest request) {
        try {
            if (request == null) {
                return ApiResponse.badRequest("El cuerpo de la peticion no puede estar vacio");
            }
            UserDTO user = userService.createUser(request);
            return ApiResponse.created(user);
        } catch (Exception e) {
            return exceptionHandler.handle(e);
        }
    }

    public ApiResponse getUser(Long id) {
        try {
            if (id == null || id <= 0) {
                return ApiResponse.badRequest("ID de usuario invalido");
            }
            UserDTO user = userService.findById(id);
            return ApiResponse.ok(user);
        } catch (Exception e) {
            return exceptionHandler.handle(e);
        }
    }

    public ApiResponse listUsers() {
        try {
            List<UserDTO> users = userService.findAll();
            return ApiResponse.ok(users);
        } catch (Exception e) {
            return exceptionHandler.handle(e);
        }
    }

    public ApiResponse updateUser(Long id, UpdateUserRequest request) {
        try {
            if (id == null || id <= 0) {
                return ApiResponse.badRequest("ID de usuario invalido");
            }
            UserDTO user = userService.update(id, request);
            return ApiResponse.ok(user);
        } catch (Exception e) {
            return exceptionHandler.handle(e);
        }
    }

    public ApiResponse deleteUser(Long id) {
        try {
            if (id == null || id <= 0) {
                return ApiResponse.badRequest("ID de usuario invalido");
            }
            userService.delete(id);
            return ApiResponse.noContent();
        } catch (Exception e) {
            return exceptionHandler.handle(e);
        }
    }
}
```

## Manejo global de errores

Un unico manejador centralizado clasifica las excepciones y produce respuestas uniformes. Cada tipo de excepcion se asigna a un codigo HTTP distinto, garantizando que el cliente reciba informacion consistente ante cualquier fallo.

```java
package com.example.presentation.controller;

import com.example.business.exception.BusinessException;
import com.example.business.exception.DuplicateResourceException;
import com.example.business.exception.NotFoundException;
import com.example.presentation.response.ApiResponse;

public class GlobalExceptionHandler {

    public ApiResponse handle(Exception e) {
        if (e instanceof NotFoundException) {
            return ApiResponse.notFound(e.getMessage());
        }
        if (e instanceof DuplicateResourceException) {
            return ApiResponse.conflict(e.getMessage());
        }
        if (e instanceof BusinessException) {
            return ApiResponse.badRequest(e.getMessage());
        }
        if (e instanceof IllegalArgumentException) {
            return ApiResponse.badRequest(e.getMessage());
        }
        return ApiResponse.serverError("Error interno del servidor");
    }
}
```

La clase `ApiResponse` unifica el formato de todas las respuestas:

```java
package com.example.presentation.response;

import java.time.LocalDateTime;

public class ApiResponse {
    private final int status;
    private final String message;
    private final Object data;
    private final LocalDateTime timestamp;

    private ApiResponse(int status, String message, Object data) {
        this.status = status;
        this.message = message;
        this.data = data;
        this.timestamp = LocalDateTime.now();
    }

    public static ApiResponse ok(Object data) {
        return new ApiResponse(200, "OK", data);
    }

    public static ApiResponse created(Object data) {
        return new ApiResponse(201, "Creado", data);
    }

    public static ApiResponse noContent() {
        return new ApiResponse(204, "Sin contenido", null);
    }

    public static ApiResponse badRequest(String message) {
        return new ApiResponse(400, message, null);
    }

    public static ApiResponse notFound(String message) {
        return new ApiResponse(404, message, null);
    }

    public static ApiResponse conflict(String message) {
        return new ApiResponse(409, message, null);
    }

    public static ApiResponse serverError(String message) {
        return new ApiResponse(500, message, null);
    }

    public int getStatus() { return status; }
    public String getMessage() { return message; }
    public Object getData() { return data; }
    public LocalDateTime getTimestamp() { return timestamp; }
}
```

## Punto de entrada de la aplicacion

El metodo `main` construye el grafo de dependencias y expone un menu textual para interactuar con los casos de uso. Esta es la unica clase que conoce la implementacion concreta de cada componente.

```java
package com.example;

import com.example.config.AppConfig;
import com.example.presentation.controller.UserController;
import com.example.presentation.controller.OrderController;
import com.example.presentation.dto.request.CreateUserRequest;
import com.example.presentation.dto.request.CreateOrderRequest;
import com.example.presentation.response.ApiResponse;
import java.util.List;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        AppConfig config = new AppConfig();
        UserController userController = config.userController();
        OrderController orderController = config.orderController();
        Scanner scanner = new Scanner(System.in);

        while (true) {
            System.out.println("1. Crear usuario");
            System.out.println("2. Listar usuarios");
            System.out.println("3. Crear orden");
            System.out.println("0. Salir");
            System.out.print("Opcion: ");

            int option = scanner.nextInt();
            scanner.nextLine();

            switch (option) {
                case 1 -> {
                    System.out.print("Nombre: ");
                    String name = scanner.nextLine();
                    System.out.print("Email: ");
                    String email = scanner.nextLine();
                    ApiResponse response = userController.createUser(new CreateUserRequest(name, email));
                    System.out.println(response.getStatus() + ": " + response.getMessage());
                }
                case 2 -> {
                    ApiResponse response = userController.listUsers();
                    System.out.println(response.getData());
                }
                case 3 -> {
                    System.out.print("User ID: ");
                    Long userId = scanner.nextLong();
                    scanner.nextLine();
                    System.out.print("Product ID: ");
                    Long productId = scanner.nextLong();
                    System.out.print("Cantidad: ");
                    int qty = scanner.nextInt();
                    scanner.nextLine();
                    CreateOrderRequest req = new CreateOrderRequest(userId,
                        List.of(new CreateOrderRequest.ItemDTO(productId, qty)));
                    ApiResponse response = orderController.createOrder(req);
                    System.out.println(response.getStatus() + ": " + response.getMessage());
                }
                case 0 -> System.exit(0);
            }
        }
    }
}
```

> La separacion entre `ApiResponse` y las excepciones permite que el controlador no tenga que construir manualmente la respuesta en cada metodo: delega el mapeo de errores al `GlobalExceptionHandler`, que decide el codigo HTTP segun el tipo de excepcion.
