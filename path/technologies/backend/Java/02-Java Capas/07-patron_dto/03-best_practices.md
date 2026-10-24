# Buenas practicas con DTOs

El uso correcto de DTOs determina en gran medida la calidad del desacoplamiento entre capas. Las siguientes practicas consolidan los patrones que separan una arquitectura mantenible de un codigo donde las capas estan acopladas aunque nominalmente separadas.

## No exponer entidades directamente

La regla mas importante del patron DTO es tambien la mas violada: las entidades de persistencia (clases anotadas con `@Entity`, tablas de base de datos) no deben salir de la capa de persistencia. Cuando un Controller retorna una entidad directamente, o cuando un Service recibe una entidad como parametro, el acoplamiento se materializa de inmediato.

```java
// MAL: exponer la entidad directamente
@GetMapping("/users/{id}")
public UserEntity getUser(@PathVariable Long id) {
    return userService.findById(id);
}

// BIEN: convertir a DTO antes de salir de la capa de presentacion
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    UserDTO user = userService.findById(id);
    return userMapper.toResponse(user);
}
```

Las consecuencias de exponer entidades directamente son multiples:

- **Campos sensibles**: una entidad JPA suele contener `passwordHash`, `recoveryToken`, `internalNotes` o campos de auditoria. Si se serializa directamente a JSON, todos esos campos se filtran al cliente.
- **Referencias circulares**: una relacion `@OneToMany` entre `OrderEntity` y `UserEntity` produce una referencia circular que causa `StackOverflowError` al serializar, o requiere anotaciones `@JsonIgnore` que contaminan la entidad con consideraciones de presentacion.
- **Lazy loading**: una entidad con carga perezosa (`FetchType.LAZY`) que se serializa fuera de una transaccion lanza `LazyInitializationException`. La solucion tipica (cambiar a `EAGER`) degrada el rendimiento global.
- **Acoplamiento de schema**: cualquier cambio en la base de datos (renombrar columna, dividir tabla) obliga a cambiar la API publica, cuando ambos deberian poder evolucionar independientemente.

```java
@Entity
@Table(name = "users")
public class UserEntity {
    @Id private Long id;
    private String name;
    @Email private String email;
    private String passwordHash;   // Campo sensible
    private String recoveryToken;  // Campo sensible

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<OrderEntity> orders; // Referencia circular potencial
}
```

> La unica excepcion justificable son los prototipos o los endpoints internos de administracion donde el coste de mantener DTOs separados no compensa el riesgo. En cualquier otro caso, la entidad no cruza la frontera de la capa de persistencia.

## DTOs inmutables

Un DTO inmutable no puede modificarse despues de su construccion. Esto elimina clases enteras de errores: nadie puede mutar un DTO despues de crearlo, no hay problemas de concurrencia en entornos multihilo, y el flujo de datos es siempre predecible.

Java 14 introdujo los `record` como la forma mas concisa de crear DTOs inmutables:

```java
public record CreateUserRequest(
    @NotBlank String name,
    @Email @NotBlank String email,
    @NotBlank @Size(min = 8) String password
) {}

public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {}
```

Para DTOs con muchos campos o con campos opcionales, el patron Builder combinado con un constructor privado proporciona la misma inmutabilidad con mejor legibilidad en el punto de creacion:

```java
public class OrderDTO {
    private final Long id;
    private final Long userId;
    private final List<OrderItemDTO> items;
    private final BigDecimal total;
    private final String status;

    private OrderDTO(Builder builder) {
        this.id = builder.id;
        this.userId = builder.userId;
        this.items = List.copyOf(builder.items); // Copia defensiva
        this.total = builder.total;
        this.status = builder.status;
    }

    public static Builder builder() {
        return new Builder();
    }

    // Solo getters, sin setters

    public static class Builder {
        private Long id;
        private Long userId;
        private List<OrderItemDTO> items = List.of();
        private BigDecimal total = BigDecimal.ZERO;
        private String status;

        public Builder id(Long id) { this.id = id; return this; }
        public Builder userId(Long userId) { this.userId = userId; return this; }
        public Builder items(List<OrderItemDTO> items) { this.items = items; return this; }
        public Builder total(BigDecimal total) { this.total = total; return this; }
        public Builder status(String status) { this.status = status; return this; }
        public OrderDTO build() { return new OrderDTO(this); }
    }
}
```

La inmutabilidad tiene implicaciones practicas: un builder debe hacer copias defensivas de las colecciones (como `List.copyOf()`) para evitar que quien invoca el builder pueda mutar la lista internamente despues de construir el objeto.

> Los `record` son siempre la primera opcion. Solo cuando se necesita un builder (muchos campos, campos opcionales con valores por defecto, herencia) se recurre a una clase tradicional con constructor privado.

## Validacion en DTOs

La validacion en los DTOs se declara mediante anotaciones de Bean Validation (JSR 380). Estas anotaciones actuan como un contrato explicito: cualquiera que lea el DTO sabe exactamente que reglas debe cumplir cada campo.

```java
public record CreateUserRequest(
    @NotBlank(message = "El nombre es obligatorio")
    @Size(min = 2, max = 100, message = "El nombre debe tener entre {min} y {max} caracteres")
    String name,

    @NotBlank
    @Email(message = "Formato de email invalido")
    String email,

    @NotBlank
    @Size(min = 8, max = 64)
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[0-9]).*$",
             message = "La contrasena debe tener mayusculas y numeros")
    String password,

    @PhoneNumber(message = "Numero de telefono invalido")
    String phone
) {}
```

**Grupos de validacion**: permiten aplicar diferentes reglas segun el contexto. El mismo DTO puede validarse con reglas distintas para creacion y para actualizacion:

```java
public interface Create {}
public interface Update {}

public class UserRequest {
    @Null(groups = Create.class)
    @NotNull(groups = Update.class)
    private Long id;

    @NotBlank(groups = Create.class)
    @Size(min = 2, max = 100)
    private String name;

    @Email
    @NotBlank(groups = Create.class)
    private String email;

    @NotBlank(groups = Create.class)
    private String password;
}
```

En el Controller, se especifica que grupo de validacion aplicar:

```java
@PostMapping
public Response create(@Validated(Create.class) @RequestBody UserRequest request) {
    // El id debe ser null, password obligatorio
}

@PutMapping("/{id}")
public Response update(@Validated(Update.class) @RequestBody UserRequest request) {
    // El id debe ser not null, password opcional
}
```

**Validacion condicional**: cuando una regla depende del valor de otro campo, se necesita un validador a nivel de clase:

```java
@Target(TYPE)
@Retention(RUNTIME)
@Constraint(validatedBy = ConditionalValidator.class)
public @interface ConditionalValidation {
    String message() default "Validacion condicional fallida";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class ConditionalValidator
        implements ConstraintValidator<ConditionalValidation, UserRequest> {

    @Override
    public boolean isValid(UserRequest request, ConstraintValidatorContext context) {
        if (request.notifyByEmail() && request.email() == null) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(
                "El email es obligatorio si se selecciona notificacion por email")
                .addPropertyNode("email")
                .addConstraintViolation();
            return false;
        }
        return true;
    }
}
```

La validacion en DTOs trata sobre la forma de los datos (formato, obligatoriedad, rangos). Las reglas de negocio (p. ej., "un usuario no puede tener mas de tres direcciones") pertenecen a la capa de servicio, no al DTO.
