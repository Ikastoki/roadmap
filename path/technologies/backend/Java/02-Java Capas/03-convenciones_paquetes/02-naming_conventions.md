# Convenciones de nomenclatura

La nomenclatura de clases en una aplicacion Java estructurada por capas no es arbitraria: cada tipo de componente tiene un sufijo estandar que comunica inmediatamente su rol en la arquitectura. Estas convenciones, heredadas de las mejores practicas de Spring y Jakarta EE, permiten a cualquier desarrollador entender la responsabilidad de una clase con solo leer su nombre, sin necesidad de inspeccionar su codigo.

## Controllers

Los controllers son el punto de entrada de la aplicacion. Reciben peticiones del exterior, delegan en los servicios y retornan respuestas formateadas.

```
UserController
OrderController
ProductController
AuthenticationController
HealthCheckController
```

Reglas de nomenclatura:
- Sufijo `Controller`, nunca abreviado (`Ctrl`, `Ctl`).
- Nombre en singular del recurso que gestiona: `UserController`, no `UsersController`.
- Si existen distintos tipos de endpoints para el mismo recurso (REST, web, admin), se puede prefijar: `AdminUserController`, `PublicUserController`.

> En aplicaciones REST, el nombre del controller suele coincidir con el nombre del recurso principal, pero no necesariamente con la entidad de persistencia. Un `OrderController` puede gestionar pedidos mientras que la entidad subyacente se llama `PurchaseEntity`.

## Services

Los servicios contienen la logica de negocio y orquestan las operaciones entre repositorios.

```
UserService
OrderService
PaymentService
EmailNotificationService
AuthenticationService
```

Reglas de nomenclatura:
- Sufijo `Service`, nunca `Svc` ni `Serv`.
- Prefijo descriptivo del dominio o caso de uso.
- Cuando un servicio orquesta operaciones de alto nivel, puede nombrarse con el caso de uso: `CreateOrderService`, `ProcessPaymentService` (patron caso de uso).

El nombre del servicio no tiene por que coincidir con el de una entidad. `EmailNotificationService` no gestiona una entidad `Email`, sino que implementa una capacidad transversal del sistema.

## Repositories

Los repositorios abstraen el acceso a datos y proporcionan una interfaz similar a una coleccion para las entidades de persistencia.

```
UserRepository
OrderRepository
ProductRepository
AuditLogRepository
SessionRepository
```

Reglas de nomenclatura:
- Sufijo `Repository`, nunca `Repo` ni `Rep`.
- Nombre en singular de la entidad que persiste: `UserRepository`, no `UsersRepository`.
- Si existe una implementacion concreta (no es la interfaz), se anade el sufijo `Impl`: `UserRepositoryImpl`, `JpaUserRepository`.
- Para repositorios de solo lectura: `UserReadRepository`, `ProductViewRepository`.

## DTOs (Data Transfer Objects)

Los DTOs transportan datos entre capas y no contienen logica de negocio. Su nomenclatura indica tanto el recurso como el contexto de uso.

### DTO generico

```
UserDTO
OrderDTO
ProductDTO
```

Regla: sufijo `DTO` (mayusculas) para el DTO generico que se usa internamente entre capas.

### Request DTOs

```
CreateUserRequest
UpdateUserRequest
LoginRequest
SearchProductRequest
PlaceOrderRequest
```

Regla: sufijo `Request` para los objetos que representan la entrada del cliente.

### Response DTOs

```
UserResponse
OrderSummaryResponse
PaymentConfirmationResponse
ErrorResponse
PagedResponse
```

Regla: sufijo `Response` para los objetos que representan la salida hacia el cliente. Si el DTO es un resumen parcial, se puede usar `Summary`: `UserSummary`.

> Con Java 14+ es recomendable usar `record` para los DTOs, lo que refuerza su naturaleza inmutable y reduce el boilerplate. Un `CreateUserRequest` como record expresa claramente que es un mero portador de datos.

## Entities

Las entidades representan el modelo de persistencia, normalmente mapeadas a tablas de base de datos mediante JPA.

```
UserEntity
OrderEntity
ProductEntity
AuditLogEntity
```

Reglas de nomenclatura:
- Sufijo `Entity` para distinguirlas del modelo de dominio.
- Si el modelo de dominio y la entidad de persistencia son la misma clase (enfoque anemico o proyectos pequenos), se omite el sufijo: `User` en lugar de `UserEntity`.
- En proyectos que siguen Domain-Driven Design, es habitual separar la entidad de dominio (`User`) de la entidad de persistencia (`UserEntity`) para mantener el dominio puro.

```
// Domain model (sin dependencias de infraestructura)
public class User {
    private UserId id;
    private Email email;
    private PersonName name;
}

// Persistence entity (mapeada a base de datos)
@Entity
@Table(name = "users")
public class UserEntity {
    @Id
    private Long id;
    private String email;
    private String name;
}
```

## Exceptions

Las excepciones personalizadas comunican condiciones de error especificas del negocio o de la capa.

```
UserNotFoundException
OrderValidationException
PaymentDeclinedException
InsufficientStockException
DuplicateEmailException
UnauthorizedAccessException
```

Reglas de nomenclatura:
- Sufijo `Exception`, siempre.
- El prefijo describe la condicion de error con la mayor precision posible.
- Se recomienda crear excepciones por categoria: `BusinessException`, `NotFoundException`, `ValidationException`, y luego extenderlas con casos concretos.

```java
public abstract class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}

public class DuplicateEmailException extends BusinessException {
    public DuplicateEmailException(String email) {
        super("El email ya esta registrado: " + email);
    }
}

public class UserNotFoundException extends BusinessException {
    public UserNotFoundException(Long id) {
        super("Usuario no encontrado con id: " + id);
    }
}
```

## Mappers

Los mappers convierten entre diferentes tipos de objetos (Entity a DTO, DTO a Entity, etc.).

```
UserMapper
OrderMapper
ProductMapper
UserEntityMapper
```

Reglas: sufijo `Mapper`. Un unico mapper por recurso suele ser suficiente. Si la conversion es bidireccional, se incluyen ambos metodos en la misma clase.

## Configuracion

Las clases de configuracion definen beans, propiedades y ajustes del framework.

```
AppConfig
SecurityConfig
DatabaseConfig
CacheConfig
CorsConfig
WebConfig
```

Reglas: sufijo `Config`. Prefijo que indica el aspecto que configuran.

## Tabla resumen

| Tipo | Sufijo | Ejemplo | Package |
|---|---|---|---|
| Controller | `Controller` | `UserController` | `presentation/controller/` |
| Service | `Service` | `UserService` | `business/service/` |
| Repository | `Repository` | `UserRepository` | `data/repository/` |
| DTO generico | `DTO` | `UserDTO` | `presentation/dto/` |
| Request DTO | `Request` | `CreateUserRequest` | `presentation/dto/` |
| Response DTO | `Response` | `UserResponse` | `presentation/dto/` |
| Entity | `Entity` | `UserEntity` | `data/entity/` |
| Exception | `Exception` | `UserNotFoundException` | `business/exception/` |
| Mapper | `Mapper` | `UserMapper` | `data/mapper/` o `presentation/dto/` |
| Config | `Config` | `AppConfig` | `config/` |

> La consistencia en la nomenclatura es mas importante que la eleccion concreta de un sufijo. Una vez adoptada una convencion, debe aplicarse sin excepciones en todo el proyecto. Las herramientas de busqueda, el analisis estatico y la revision de codigo se benefician enormemente de esta uniformidad.
