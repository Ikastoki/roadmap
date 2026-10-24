# Object Mother

El patron Object Mother centraliza la creacion de objetos de prueba en una unica clase o conjunto de clases, proporcionando metodos estaticos que devuelven instancias preconfiguradas para diferentes escenarios. A diferencia del Test Data Builder, que construye objetos paso a paso mediante una interfaz fluida, Object Mother ofrece objetos completos listos para usar.

## Fabrica centralizada

Una Object Mother es esencialmente una fabrica de objetos de prueba. En lugar de construir cada objeto en el test o tener un builder por cada clase, se definen metodos que devuelven instancias con configuraciones tipicas para distintos escenarios.

```java
public class UserMother {

    public static User anActiveUser() {
        return new User("user-1", "Active User", "active@example.com", 30, true);
    }

    public static User anInactiveUser() {
        return new User("user-2", "Inactive User", "inactive@example.com", 25, false);
    }

    public static User aPremiumUser() {
        return new User("user-3", "Premium User", "premium@example.com", 35, true);
    }

    public static User aMinorUser() {
        return new User("user-4", "Minor User", "minor@example.com", 15, true);
    }

    public static List<User> multipleUsers(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> new User("user-" + i, "User " + i,
                "user" + i + "@example.com", 20 + i, true))
            .toList();
    }
}
```

## Metodos estaticos

Los metodos de Object Mother son estaticos y devuelven objetos completamente construidos. El test los usa directamente sin necesidad de instanciar fabricas ni configurar builders:

```java
class NotificationServiceTest {

    private final NotificationService service = new NotificationService();

    @Test
    void shouldSendWelcomeEmailToActiveUser() {
        User user = UserMother.anActiveUser();

        NotificationResult result = service.sendWelcome(user);

        assertTrue(result.isSent());
    }

    @Test
    void shouldNotSendPromotionsToInactiveUser() {
        User user = UserMother.anInactiveUser();

        NotificationResult result = service.sendPromotion(user);

        assertFalse(result.isSent());
    }
}
```

## Ventajas

- **Reutilizacion**: los mismos objetos de prueba se usan en multiples tests, evitando duplicacion
- **Consistencia**: todos los tests usan los mismos datos base, reduciendo la divergencia entre escenarios similares
- **Legibilidad**: el test expresa la intencion mediante el nombre del metodo: `UserMother.aPremiumUser()` es mas claro que construir un usuario con campos especificos
- **Mantenimiento**: si la estructura de un objeto cambia, solo hay que modificar la Object Mother, no todos los tests

## Desventajas

- **Acoplamiento**: todos los tests dependen de la misma clase, creando una dependencia centralizada
- **Fragilidad**: cambios en la Object Mother pueden afectar a multiples tests inesperadamente
- **Inflacion**: con el tiempo la Object Mother tiende a crecer con variaciones de cada objeto, volviendose dificil de mantener
- **Falta de flexibilidad**: los objetos vienen preconfigurados; si un test necesita una variacion especifica, debe o bien modificar el objeto devuelto o crear una nueva entrada en la madre

> Object Mother es adecuado para objetos simples o cuando los escenarios de prueba estan bien definidos y son estables. Para objetos con muchas variaciones, el Builder Pattern suele ser mejor opcion.

## Combinacion con Builder

Una practica recomendada es combinar ambos patrones: usar Object Mother para los escenarios mas comunes y Builder para personalizar casos especificos.

```java
public class OrderMother {

    public static Order aStandardOrder() {
        return OrderBuilder.anOrder().build();
    }

    public static Order aRushOrder() {
        return OrderBuilder.anOrder()
            .withPriority(Priority.HIGH)
            .withShippingMethod(ShippingMethod.EXPRESS)
            .build();
    }

    public static Order anInternationalOrder() {
        return OrderBuilder.anOrder()
            .withCurrency("USD")
            .withDestinationCountry("US")
            .build();
    }

    public static Order anOrderWithAmount(double amount) {
        return OrderBuilder.anOrder()
            .withAmount(amount)
            .build();
    }
}
```

Esta combinacion ofrece lo mejor de ambos mundos: la simplicidad de Object Mother para el caso comun y la flexibilidad de Builder para el caso particular.

```java
@Test
void shouldCalculateShippingCostForRushOrder() {
    Order order = OrderMother.aRushOrder();

    double shipping = shippingService.calculate(order);

    assertEquals(25.0, shipping, 0.001);
}

@Test
void shouldHandleFreeShippingForLargeOrders() {
    Order order = OrderMother.anOrderWithAmount(500.0);

    double shipping = shippingService.calculate(order);

    assertEquals(0.0, shipping, 0.001);
}
```
