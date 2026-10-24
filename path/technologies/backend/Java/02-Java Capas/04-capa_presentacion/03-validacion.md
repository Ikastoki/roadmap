# Validacion de entrada

Toda entrada que proviene del exterior debe ser validada antes de ser procesada. Un sistema que acepta datos sin verificarlos esta expuesto a errores de integridad, vulnerabilidades de seguridad y comportamientos inesperados. La validacion actua como la primera barrera defensiva: rechaza datos malformados antes de que lleguen a la capa de negocio, donde su procesamiento podria ser costoso o peligroso.

En la arquitectura de capas, la validacion ocurre en dos niveles. En la capa de presentacion se valida la forma de los datos (campos obligatorios, formatos, rangos). En la capa de negocio se validan las reglas del dominio (unicidad, consistencia, autorizacion). Mezclar ambos niveles produce codigo dificil de mantener y de testear.

## Bean Validation (JSR 380)

Bean Validation es el estandar de Java para la validacion declarativa de objetos. Definido por la JSR 380 (Jakarta Bean Validation 3.0), permite anadir restricciones a los campos de un DTO mediante anotaciones, y el framework evalua esas restricciones automaticamente en el momento de la validacion.

```java
public class CreateUserRequest {
    @NotBlank(message = "El nombre es obligatorio")
    @Size(min = 2, max = 100, message = "El nombre debe tener entre 2 y 100 caracteres")
    private String name;

    @NotBlank
    @Email(message = "Formato de email invalido")
    private String email;

    @NotNull
    @Positive(message = "La edad debe ser un numero positivo")
    private Integer age;
}
```

La validacion se dispara anotando el parametro del Controller con `@Valid` (Jakarta) o `@Validated` (Spring). Si alguna restriccion se incumple, el framework lanza una excepcion que se traduce automaticamente en una respuesta HTTP 400 Bad Request.

```java
public Response createUser(@Valid CreateUserRequest request) {
    UserDTO user = userService.createUser(request);
    return Response.created(user).build();
}
```

## Anotaciones comunes

El estandar define un conjunto de anotaciones que cubren la mayoria de los casos de validacion de entrada:

| Anotacion | Funcion | Ejemplo |
|---|---|---|
| `@NotNull` | El campo no puede ser `null` | `@NotNull Long id` |
| `@NotBlank` | La cadena no puede ser `null`, vacia o solo espacios | `@NotBlank String name` |
| `@NotEmpty` | La coleccion/cadena no puede estar vacia | `@NotEmpty List<String> roles` |
| `@Size(min, max)` | Longitud de cadena o tamano de coleccion | `@Size(min=3, max=20) String username` |
| `@Min(value)` | Valor numerico minimo (inclusive) | `@Min(1) int quantity` |
| `@Max(value)` | Valor numerico maximo (inclusive) | `@Max(10000) BigDecimal amount` |
| `@Email` | Formato de direccion de email | `@Email String email` |
| `@Pattern(regexp)` | Expresion regular | `@Pattern("^\\+?\\d{7,15}$") String phone` |
| `@Past` | Fecha en el pasado | `@Past LocalDate birthDate` |
| `@PastOrPresent` | Fecha pasada o actual | `@PastOrPresent LocalDate startDate` |
| `@Future` | Fecha en el futuro | `@Future LocalDate expirationDate` |
| `@FutureOrPresent` | Fecha futura o actual | `@FutureOrPresent LocalDate deadline` |
| `@Positive` | Numero estrictamente positivo | `@Positive BigDecimal price` |
| `@PositiveOrZero` | Numero positivo o cero | `@PositiveOrZero int stock` |
| `@Negative` | Numero estrictamente negativo | `@Negative BigDecimal discount` |
| `@NegativeOrZero` | Numero negativo o cero | `@NegativeOrZero int adjustment` |

> La anotacion `@NotBlank` es preferible a `@NotNull` en campos de tipo `String` porque cubre los casos de cadena vacia y de solo espacios blancos. Para colecciones, `@NotEmpty` es la equivalente.

## Validadores personalizados

Cuando las anotaciones estandar no cubren una regla de validacion especifica (un formato de telefono, un identificador fiscal, un patron de contrasena), se puede crear un validador personalizado. El proceso requiere dos elementos: una anotacion que define la restriccion y una clase que implementa la logica de validacion.

```java
// Anotacion personalizada
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@Constraint(validatedBy = PhoneNumberValidator.class)
public @interface PhoneNumber {
    String message() default "Numero de telefono invalido";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Implementacion del validador
public class PhoneNumberValidator implements ConstraintValidator<PhoneNumber, String> {
    private static final String PHONE_PATTERN = "^\\+?[1-9]\\d{1,14}$";

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true; // @NotNull se encarga de este caso
        }
        return value.matches(PHONE_PATTERN);
    }
}
```

Uso del validador personalizado:

```java
public class UpdateUserRequest {
    @PhoneNumber(message = "El telefono debe tener formato internacional valido")
    private String phone;
}
```

El validador personalizado permite encapsular reglas complejas de validacion (expresiones regulares, comprobaciones contra bases de datos externas, algoritmos de checksum) y reutilizarlas en todos los DTOs que las necesiten.

## Grupos de validacion

Un mismo DTO puede requerir reglas de validacion diferentes segun el contexto. Por ejemplo, al crear un usuario el campo `id` debe ser `null` (lo genera la base de datos), pero al actualizarlo debe ser `@NotNull`. Los grupos de validacion permiten definir estas variaciones sin duplicar el DTO.

```java
// Interfaces de marcado para los grupos
public interface Create {}
public interface Update {}

public class UserRequest {
    @Null(groups = Create.class, message = "El ID no debe enviarse en creacion")
    @NotNull(groups = Update.class, message = "El ID es obligatorio en actualizacion")
    private Long id;

    @NotBlank(groups = {Create.class, Update.class})
    @Size(min = 2, max = 100)
    private String name;

    @NotBlank(groups = Create.class)
    @Email
    private String email;
}
```

En el Controller, se especifica que grupo debe validarse en cada operacion:

```java
public Response createUser(@Validated(Create.class) UserRequest request) {
    UserDTO user = userService.createUser(request);
    return Response.created(user).build();
}

public Response updateUser(@Validated(Update.class) UserRequest request) {
    UserDTO user = userService.updateUser(request);
    return Response.ok(user).build();
}
```

Los grupos de validacion evitan la proliferacion de DTOs especificos para cada operacion y mantienen las reglas de validacion centralizadas en un unico lugar. Sin embargo, cuando las diferencias entre contextos son grandes (campos completamente distintos), es preferible usar DTOs separados antes que un unico DTO con grupos complejos.
