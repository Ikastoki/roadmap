# DTO (Data Transfer Object)

Un DTO es un objeto cuyo unico proposito es transportar datos entre capas de la aplicacion. No contiene logica de negocio, no implementa comportamientos, no valida reglas del dominio. Es, esencialmente, un contenedor tipado de datos con una estructura bien definida que actua como contrato entre la capa de presentacion y el exterior o entre capas internas.

La existencia del DTO responde a una necesidad concreta: las entidades de dominio y de persistencia tienen una forma que responde a la logica interna de la aplicacion, no a lo que el cliente necesita ver o enviar. Exponer esas entidades directamente al exterior crea un acoplamiento rigido que propaga cambios de la base de datos a la API y viceversa.

## Por que no usar entidades directamente

Entregar una entidad JPA directamente como respuesta de un endpoint tiene tres consecuencias negativas inmediatas:

- **Exposicion de la estructura interna de la base de datos**: cualquier cambio en el esquema de tablas (renombrar una columna, dividir una tabla, introducir una relacion nueva) se refleja automaticamente en la respuesta de la API, obligando a los clientes a adaptarse. La base de datos deja de ser un detalle de implementacion y pasa a ser parte del contrato publico.
- **Acoplamiento entre capas**: la capa de presentacion depende directamente de las clases de persistencia. Un cambio en la entidad (un nuevo campo lazy, una relacion que cambia de FetchType) puede romper la serializacion JSON. El resultado es que modificar la base de datos requiere modificar la API, y modificar la API requiere modificar la base de datos.
- **Riesgo de seguridad**: las entidades suelen contener campos sensibles que no deberian exponerse (contrasenas hash, tokens internos, flags de administrador, timestamps de auditoria). Un error en la configuracion de serializacion (o simplemente olvidar anadir `@JsonIgnore`) expone esos datos al cliente.

```java
// MAL: expone la entidad directamente
public class UserController {
    public UserEntity getUser(Long id) { // UserEntity tiene password, internalNotes, etc.
        return userRepository.findById(id).orElseThrow();
    }
}

// BIEN: el DTO solo expone lo necesario
public record UserResponse(Long id, String name, String email, LocalDateTime createdAt) {}
```

## Tipos de DTO

No todos los DTOs tienen la misma funcion. Segun su uso y el punto de la arquitectura donde se emplean, se clasifican en:

- **Request DTO**: representa los datos que el cliente envia al servidor. Contiene exactamente los campos que el cliente puede proporcionar, ni uno mas. Ejemplos: `CreateUserRequest`, `UpdateUserRequest`, `LoginRequest`. Suele incluir anotaciones de validacion (Bean Validation).
- **Response DTO**: representa los datos que el servidor devuelve al cliente. Puede contener campos calculados o agregados que no existen en la base de datos. Ejemplos: `UserResponse`, `OrderSummary`.
- **Internal DTO**: se usa para la comunicacion entre capas internas (por ejemplo, entre Service y Repository, o entre Service y un adaptador externo). Es mas completo que un DTO de presentacion y puede incluir campos que el cliente no debe ver.
- **Summary DTO**: una version reducida de un Response DTO, usada en listados y busquedas donde no se necesita la informacion completa del recurso. Ejemplo: `UserSummary` con solo `id`, `name` y `avatarUrl`.

## DTOs con Record (Java 14+)

A partir de Java 14, los `record` proporcionan la sintaxis mas concisa para definir DTOs inmutables. Un `record` genera automaticamente el constructor, los getters, `equals()`, `hashCode()` y `toString()` basados en los componentes declarados.

```java
public record UserDTO(Long id, String name, String email) {}

public record CreateUserRequest(String name, String email, String password) {}

public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt,
    List<String> roles
) {}
```

La inmutabilidad es una propiedad deseable en los DTOs: un objeto que no puede modificarse despues de su creacion elimina problemas de aliasing, simplifica el razonamiento sobre el flujo de datos y evita mutaciones accidentales en cualquier punto de la cadena de llamadas.

> Los `record` son ideales para DTOs simples con pocos campos. Cuando el DTO crece en complejidad (campos opcionales, valores por defecto, construccion condicional), el Builder Pattern es una alternativa mas adecuada.

## Builder Pattern

Cuando un DTO tiene muchos campos (tipicamente mas de cuatro), campos opcionales, o requiere logica de construccion, el patron Builder ofrece una API mas legible y segura que un constructor con multiples parametros.

```java
public class UserDTO {
    private final Long id;
    private final String name;
    private final String email;
    private final LocalDateTime createdAt;
    private final String phone;

    private UserDTO(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.email = builder.email;
        this.createdAt = builder.createdAt;
        this.phone = builder.phone;
    }

    public static class Builder {
        private Long id;
        private String name;
        private String email;
        private LocalDateTime createdAt;
        private String phone;

        public Builder id(Long id) { this.id = id; return this; }
        public Builder name(String name) { this.name = name; return this; }
        public Builder email(String email) { this.email = email; return this; }
        public Builder createdAt(LocalDateTime createdAt) { this.createdAt = createdAt; return this; }
        public Builder phone(String phone) { this.phone = phone; return this; }
        public UserDTO build() { return new UserDTO(this); }
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public String getPhone() { return phone; }
}
```

El Builder permite construir el objeto paso a paso, haciendo explicito que campos se estan asignando. Ademas, facade el problema de la retrocompatibilidad: anadir un nuevo campo al DTO solo requiere anadirlo al Builder sin romper el codigo existente que usa el Builder sin ese campo.

## DTOs inmutables

Tanto los `record` como los DTOs construidos con Builder deben ser inmutables. Esto significa:

- Todos los campos son `final`.
- No existen setters.
- Las colecciones devueltas por los getters son copias inmodificables (`Collections.unmodifiableList()`, `List.copyOf()`).
- Si el DTO contiene objetos mutables, se devuelven copias defensivas en los getters.

La inmutabilidad garantiza que una vez que el DTO abandona la capa que lo creo, ningun otro componente puede alterar su contenido. Esto es especialmente importante cuando el DTO cruza limites arquitectonicos: un Response DTO que viaja desde el Service hasta el Controller, y de ahi al serializador JSON, no debe ser modificado en ninguna estacion del trayecto.
