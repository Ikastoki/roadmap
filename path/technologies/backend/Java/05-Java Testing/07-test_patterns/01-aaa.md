# AAA (Arrange-Act-Assert)

El patron AAA estructura los tests unitarios en tres fases claramente diferenciadas: Arrange (preparar), Act (ejecutar) y Assert (verificar). Esta separacion no es un mero convenio estetico: responde a la necesidad de que un test sea legible como una historia breve en la que cada seccion tiene un proposito unico y reconocible.

## Arrange (preparar)

En la fase Arrange se construye el escenario del test: se crean los objetos necesarios, se configuran las dependencias, se inicializan los datos de entrada. Todo lo que el test necesita para ejecutarse debe estar preparado aqui, sin mezclar preparacion con ejecucion o verificacion.

```java
@Test
void shouldCalculateTotalWithTax() {
    // Arrange
    TaxCalculator calculator = new TaxCalculator(0.21);
    Invoice invoice = new Invoice(List.of(
        new Item("Laptop", 1000.0),
        new Item("Mouse", 50.0)
    ));
```

La fase Arrange debe responder a la pregunta: "que necesito para que el comportamiento que voy a probar sea posible?". Si la preparacion es excesivamente larga, puede ser sintoma de que el codigo bajo test tiene demasiadas dependencias o de que el test esta probando demasiadas cosas a la vez.

> Un Arrange extenso es una bandera roja: sugiere que el codigo bajo test tiene responsabilidades multiples o acoplamiento excesivo.

## Act (ejecutar)

La fase Act contiene la invocacion unica al metodo o funcionalidad bajo prueba. Una sola linea de codigo que ejecuta la operacion que se desea verificar. Si la fase Act requiere mas de una o dos lineas, probablemente se esta probando un escenario demasiado complejo para un unico test.

```java
    // Act
    double total = calculator.calculateTotal(invoice);
```

La regla de oro es: solo debe haber una operacion Act por test. Si se ejecutan varias acciones antes de verificar, no queda claro cual de ellas produjo el resultado que se esta comprobando.

## Assert (verificar)

En la fase Assert se verifican los resultados de la operacion ejecutada. Se comprueba que el valor devuelto es el esperado, que el estado del objeto ha cambiado correctamente o que se invocaron las dependencias adecuadas.

```java
    // Assert
    assertEquals(1270.5, total, 0.001);
}
```

Cada assert debe verificar un aspecto concreto del resultado. Si se necesitan multiples asserts, deben estar todos relacionados con el mismo comportamiento. Asserts sobre propiedades no relacionadas indican que el test deberia dividirse.

## Ejemplo completo

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class PaymentProcessorTest {

    @Test
    void shouldApprovePaymentWhenBalanceIsSufficient() {
        // Arrange
        Account account = new Account("ES123456789", 500.0);
        PaymentProcessor processor = new PaymentProcessor();
        double amount = 300.0;

        // Act
        PaymentResult result = processor.processPayment(account, amount);

        // Assert
        assertTrue(result.isApproved());
        assertEquals(200.0, account.getBalance(), 0.001);
        assertEquals("ES123456789", result.getAccountId());
    }

    @Test
    void shouldRejectPaymentWhenBalanceIsInsufficient() {
        // Arrange
        Account account = new Account("ES123456789", 100.0);
        PaymentProcessor processor = new PaymentProcessor();
        double amount = 300.0;

        // Act
        PaymentResult result = processor.processPayment(account, amount);

        // Assert
        assertFalse(result.isApproved());
        assertEquals(100.0, account.getBalance(), 0.001);
        assertEquals("Saldo insuficiente", result.getErrorMessage());
    }
}
```

## Separacion clara

La separacion visual entre las tres fases puede lograrse con comentarios o simplemente con lineas en blanco. Lo importante es que la intencion de cada bloque sea evidente. Algunos equipos prefieren la convencion de separar con saltos de linea sin comentarios, confiando en que la estructura es reconocible por si misma:

```java
@Test
void shouldReturnDiscountedPriceForLoyalCustomers() {
    Customer customer = new Customer("loyal", 5);
    PricingService service = new PricingService();

    double price = service.calculatePrice(100.0, customer);

    assertEquals(90.0, price, 0.001);
}
```

La consistencia en el estilo de separacion dentro del equipo es mas importante que la convencion concreta elegida.
