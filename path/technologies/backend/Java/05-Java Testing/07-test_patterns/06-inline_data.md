# Inline Data

El patron Inline Data consiste en definir los datos de prueba directamente en el cuerpo del test, sin recurrir a fabricas externas, builders o ficheros de datos. Es el patron mas simple y directo, adecuado para tests donde los datos son pocos, especificos del escenario y no se repiten en otros tests.

## Datos inline en el test

Los datos se definen como literales dentro del metodo de test. No hay abstracciones ni redirecciones: el test es autocontenido.

```java
@Test
void shouldCalculateBmi() {
    double weight = 70.0;
    double height = 1.75;

    double bmi = BmiCalculator.calculate(weight, height);

    assertEquals(22.86, bmi, 0.01);
}
```

La ventaja principal es la claridad inmediata: cualquier persona que lea el test ve exactamente que datos se estan usando y cual es el resultado esperado. No necesita navegar a otras clases para entender el escenario.

## @MethodSource

Cuando un mismo escenario debe probarse con multiples conjuntos de datos, JUnit 5 proporciona `@MethodSource` para parametrizar tests sin sacrificar la claridad de los datos inline.

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;
import java.util.stream.Stream;

class DiscountCalculatorTest {

    @ParameterizedTest
    @MethodSource("discountScenarios")
    void shouldApplyCorrectDiscount(double amount, double expected) {
        DiscountCalculator calculator = new DiscountCalculator();

        double result = calculator.calculate(amount);

        assertEquals(expected, result, 0.001);
    }

    static Stream<Arguments> discountScenarios() {
        return Stream.of(
            Arguments.of(50.0, 50.0),    // sin descuento
            Arguments.of(100.0, 100.0),  // sin descuento (limite)
            Arguments.of(150.0, 135.0),  // 10% descuento
            Arguments.of(500.0, 400.0),  // 20% descuento
            Arguments.of(0.0, 0.0)       // cero
        );
    }
}
```

El metodo fuente devuelve un `Stream` de argumentos, donde cada elemento es un caso de prueba. Los datos estan inline en el metodo fuente, manteniendo la legibilidad y evitando la duplicacion de codigo de test.

> @MethodSource combina la simplicidad del inline data con la potencia de los parametrized tests. Es la herramienta ideal para probar variaciones de un mismo comportamiento.

## Records para datos de prueba

Java 14 introdujo los records, que son especialmente utiles como contenedores de datos en tests. Su sintaxis concisa y su inmutabilidad por defecto los hacen ideales para representar conjuntos de datos de prueba.

```java
record DiscountScenario(double amount, double expectedDiscount, String description) {}

class DiscountServiceTest {

    @ParameterizedTest
    @MethodSource("scenarios")
    void shouldApplyDiscount(DiscountScenario scenario) {
        DiscountService service = new DiscountService();

        double result = service.apply(scenario.amount());

        assertEquals(scenario.expectedDiscount(), result, 0.001);
    }

    static Stream<DiscountScenario> scenarios() {
        return Stream.of(
            new DiscountScenario(100.0, 100.0, "por debajo del umbral"),
            new DiscountScenario(200.0, 180.0, "10% de descuento"),
            new DiscountScenario(500.0, 400.0, "20% de descuento"),
            new DiscountScenario(0.0, 0.0, "importe cero")
        );
    }
}
```

Los records eliminan la necesidad de clases auxiliares con getters, equals y hashCode. Al ser tipos inmutables, los datos de prueba no pueden ser modificados accidentalmente dentro del test.

## Inline data vs Object Mother vs Builder

| Aspecto | Inline Data | Object Mother | Builder |
|---------|-------------|---------------|---------|
| Datos por test | Especificos y locales | Predefinidos y globales | Configurables |
| Repeticion entre tests | Alta (cada test define sus datos) | Baja (los datos estan centralizados) | Media |
| Legibilidad | Maxima (datos visibles) | Buena (nombres descriptivos) | Buena (interfaz fluida) |
| Mantenimiento | Dificil si los datos cambian | Centralizado | Distribuido |

La eleccion depende del contexto. Inline data es adecuado para datos simples y escenarios unicos. Object Mother y Builder son mejores cuando los mismos datos se repiten en multiples tests o cuando los objetos son complejos.

```java
// Inline data es perfecto para tests simples
@Test
void shouldFormatDate() {
    LocalDate date = LocalDate.of(2024, 12, 25);

    String formatted = DateFormatter.format(date);

    assertEquals("25/12/2024", formatted);
}

// Pero para objetos complejos, el builder es preferible
@Test
void shouldCreateOrderWithMultipleItems() {
    Order order = OrderBuilder.anOrder()
        .withItems(List.of(
            new ItemBuilder().withPrice(100.0).build(),
            new ItemBuilder().withPrice(200.0).build()
        ))
        .build();
    // ...
}
```
