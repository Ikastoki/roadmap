# Builder Pattern en tests

Cuando los objetos de dominio tienen multiples campos, algunos obligatorios y otros opcionales, construir instancias dentro de los tests se vuelve repetitivo y propenso a errores. El Test Data Builder resuelve este problema proporcionando una interfaz fluida que hace explicito que valores se estan configurando y cuales se estan dejando por defecto.

## Test Data Builder

Un Test Data Builder es una clase separada que construye instancias de un objeto de dominio especifico. Su proposito exclusivo es facilitar la creacion de objetos en los tests, y por tanto no forma parte del codigo de produccion.

Estructura tipica:
- Campos con valores por defecto sensatos
- Metodos setter que devuelven `this` (fluent interface)
- Un metodo `build()` que construye la instancia final

```java
public class UserBuilder {
    private String id = "default-id";
    private String name = "Default Name";
    private String email = "default@example.com";
    private int age = 25;
    private boolean active = true;

    public UserBuilder withId(String id) {
        this.id = id;
        return this;
    }

    public UserBuilder withName(String name) {
        this.name = name;
        return this;
    }

    public UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserBuilder withAge(int age) {
        this.age = age;
        return this;
    }

    public UserBuilder inactive() {
        this.active = false;
        return this;
    }

    public User build() {
        return new User(id, name, email, age, active);
    }
}
```

## Fluent interface

La interfaz fluida permite encadenar llamadas de forma natural, creando una sintaxis que se lee como una frase:

```java
User user = new UserBuilder()
    .withName("Alice")
    .withEmail("alice@example.com")
    .withAge(30)
    .build();
```

Cada metodo devuelve el propio builder, lo que permite concatenar configuraciones sin necesidad de variables intermedias.

## Defaults

La clave de un buen builder esta en los valores por defecto. Estos deben ser valores validos pero claramente artificiales para que sea evidente cuando no se ha configurado un campo intencionadamente.

```java
public class OrderBuilder {
    private String orderId = "ORD-000";
    private double amount = 100.0;
    private String currency = "EUR";
    private LocalDate date = LocalDate.of(2024, 1, 1);
    private Customer customer = new CustomerBuilder().build();
    private List<Item> items = List.of(new ItemBuilder().build());
```

> Los valores por defecto hacen que el test se centre solo en lo que importa para el escenario actual. Un test que prueba el calculo de IVA no deberia necesitar especificar el nombre del cliente.

## Sobreescribir valores

Cuando un escenario necesita un valor especifico, se sobreescribe el campo correspondiente:

```java
@Test
void shouldRecalculateTaxWhenCurrencyChanges() {
    Order order = new OrderBuilder()
        .withAmount(100.0)
        .withCurrency("USD")
        .build();

    TaxService service = new TaxService();
    double tax = service.calculateTax(order);

    assertEquals(21.0, tax, 0.001);
}
```

## Ejemplo UserBuilder en contexto

```java
class UserServiceTest {

    private final UserRepository repository = new FakeUserRepository();
    private final UserService service = new UserService(repository);

    @Test
    void shouldActivateUser() {
        User user = new UserBuilder()
            .withName("John Doe")
            .withEmail("john@example.com")
            .inactive()
            .build();

        repository.save(user);
        service.activate(user.getId());

        User activated = repository.findById(user.getId()).get();
        assertTrue(activated.isActive());
    }

    @Test
    void shouldRegisterMultipleUsers() {
        User alice = new UserBuilder().withName("Alice").build();
        User bob = new UserBuilder().withName("Bob").build();
        User charlie = new UserBuilder().withName("Charlie").build();

        service.register(alice);
        service.register(bob);
        service.register(charlie);

        assertEquals(3, repository.count());
    }
}
```

## Variaciones

Existen otras variantes del patron:
- **Constructor nombrado**: metodos estaticos como `UserBuilder.anUser().withName(...)` para mejorar la legibilidad
- **Constructor recursive**: builders que a su vez usan builders para objetos anidados
- **Builder con estados predefinidos**: metodos como `anAdultUser()` o `aMinorUser()` que configuran grupos de campos

```java
public class UserBuilder {

    public static UserBuilder anUser() {
        return new UserBuilder();
    }

    public static UserBuilder anAdultUser() {
        return anUser().withAge(30);
    }

    public static UserBuilder aMinorUser() {
        return anUser().withAge(15);
    }
    // ...
}
```

El Test Data Builder es especialmente valioso en proyectos con modelos de dominio ricos, donde la construccion de objetos requiere multiples parametros y la legibilidad de los tests es prioritaria.
