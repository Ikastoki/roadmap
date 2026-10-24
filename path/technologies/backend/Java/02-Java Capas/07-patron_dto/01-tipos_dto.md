# Tipos de DTO

Un DTO (Data Transfer Object) no es un concepto unico: existen variantes especializadas segun la capa entre la que transporta informacion y el origen de los datos que contiene. Cada tipo responde a un proposito distinto y lleva consigo reglas de validacion, composicion y ciclo de vida diferentes. Mezclarlos es una de las causas mas frecuentes de acoplamiento innecesario entre capas.

## Request DTO

El Request DTO modela la informacion que el cliente envia al servidor. Su diseno refleja exclusivamente lo que el cliente puede y debe proporcionar, no la estructura interna del sistema. Esto implica que un Request DTO:

- Solo contiene campos que el cliente puede enviar. Nunca incluye campos calculados, identificadores generados automaticamente o informacion de auditoria.
- Lleva anotaciones de validacion de entrada (Bean Validation) que verifican formato, obligatoriedad y rangos en el momento de la llegada.
- No incluye logica de negocio ni referencias a entidades de persistencia.

```java
public record CreateUserRequest(
    @NotBlank String name,
    @Email @NotBlank String email,
    @Size(min = 8, max = 64) String password,
    @PhoneNumber String phone
) {}
```

```java
public class UpdateUserRequest {
    @NotBlank(groups = Update.class)
    private Long id;

    @Size(min = 2, max = 100)
    private String name;

    @Email
    private String email;

    // getters y setters necesarios para la deserializacion
}
```

Un Request DTO suele tener una vida corta: se crea al deserializar la peticion, se valida, se convierte a un modelo interno y se descarta. No deberia circular por las capas internas de la aplicacion.

## Response DTO

El Response DTO modela la informacion que el servidor devuelve al cliente. A diferencia del Request DTO, puede contener campos calculados, datos agregados de multiples fuentes y valores que no existen como columnas en la base de datos.

```java
public record UserResponse(
    Long id,
    String name,
    String email,
    String profileUrl,
    LocalDateTime createdAt,
    boolean isActive
) {}
```

El Response DTO tambien puede incluir campos que son el resultado de operaciones de negocio:

```java
public record OrderResponse(
    Long id,
    List<OrderItemResponse> items,
    BigDecimal subtotal,
    BigDecimal tax,
    BigDecimal total,
    String status
) {
    public OrderResponse {
        // Validacion interna: el total debe ser consistente
        BigDecimal expected = subtotal.add(tax);
        if (total.compareTo(expected) != 0) {
            throw new IllegalArgumentException("Total inconsistente con subtotal e impuestos");
        }
    }
}
```

> Un error comun es usar la misma clase para Request y Response. Aunque a veces los campos coinciden, mantenerlas separadas permite que cada una evolucione independientemente: el cliente puede enviar campos que no se devuelven (como passwords) y recibir campos que no envio (como fechas de creacion).

## Internal DTO

El Internal DTO se utiliza para la comunicacion entre capas internas de la aplicacion, tipicamente entre la capa de servicio y la capa de presentacion, o entre diferentes servicios dentro de la misma aplicacion. Es mas completo que los DTOs externos porque transporta toda la informacion que las capas internas necesitan, sin las restricciones de la interfaz publica.

```java
public class InternalUserDTO {
    private Long id;
    private String name;
    private String email;
    private String passwordHash; // Campo interno, no se expone al cliente
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private int loginAttempts;   // Solo para logica interna
    private String recoveryToken; // Solo para flujos internos
}
```

El Internal DTO actua como lingua franca entre las capas: la capa de persistencia devuelve entidades que el servicio convierte a Internal DTO, y la capa de presentacion convierte el Internal DTO a Response DTO antes de enviarlo al cliente. Esta separacion evita que la estructura de la base de datos o las necesidades internas se filtren a la API publica.

## Domain DTO vs Persistence DTO

La distincion mas sutil y a la vez mas importante es la que separa el DTO de dominio del DTO de persistencia. Ambos viajan entre capas, pero representan modelos distintos.

**Domain DTO**: refleja el modelo de dominio, es decir, los conceptos del negocio independientemente de como se almacenan. Un Domain DTO puede contener datos de multiples entidades de persistencia, valores calculados y estructuras que no existen en la base de datos.

```java
public record UserDomainDTO(
    UserId id,
    PersonName name,
    EmailAddress email,
    PasswordHash password,
    UserStatus status,
    List<Role> roles
) {
    // El dominio usa tipos propios (value objects), no tipos primitivos
}
```

**Persistence DTO**: refleja la estructura de la base de datos. Mapea directamente a tablas y columnas. Es el modelo que usan los repositorios y la capa de persistencia.

```java
@Entity
@Table(name = "users")
public class UserEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "email", unique = true, nullable = false)
    private String email;

    @Column(name = "password_hash")
    private String passwordHash;

    @Enumerated(EnumType.STRING)
    private UserStatus status;
}
```

La separacion entre ambos permite que el modelo de dominio no se contamine con consideraciones de persistencia (indices, claves foraneas, estrategias de carga) y que el esquema de base de datos pueda cambiar sin afectar al dominio. En aplicaciones donde ambos modelos convergen (un DTO que es a la vez entidad JPA y modelo de dominio), cualquier cambio en la base de datos obliga a cambiar la logica de negocio, y cualquier cambio en el negocio obliga a cambiar el esquema.

> En la practica, muchos proyectos optan por un solo DTO interno que cumple ambas funciones cuando la aplicacion es pequena o cuando el mapeo entre dominio y persistencia es directo. Esta decision es aceptable siempre que se sea consciente del acoplamiento que introduce y se tenga un plan para separarlos cuando la complejidad lo exija.
