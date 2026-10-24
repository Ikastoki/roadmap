# Pasos de TDD

El ciclo de TDD se ejecuta en incrementos microscopicos. Cada iteracion dura entre segundos y pocos minutos, y cada paso tiene una intencion clara y un criterio de salida definido. La disciplina consiste en no saltarse ningun paso y en no anadir codigo que no este justificado por el test actual.

## Red: escribir un test que falle

El primer paso consiste en escribir un test que describa el siguiente comportamiento que deseamos implementar. En este momento el test debe fallar porque la funcionalidad aun no existe. Si el test pasa sin escribir codigo de produccion, indica que el comportamiento ya esta cubierto o que el test es defectuoso.

Criterios para un buen test en fase Red:
- Prueba una sola responsabilidad o comportamiento
- Es independiente de otros tests
- Produce un fallo claro y diagnostico
- No asume detalles de implementacion interna

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class FizzBuzzTest {

    @Test
    void shouldReturnFizzWhenNumberIsDivisibleByThree() {
        FizzBuzz fizzbuzz = new FizzBuzz();
        assertEquals("Fizz", fizzbuzz.of(3));
    }
}
```

Al ejecutar este test obtenemos un error de compilacion porque la clase `FizzBuzz` no existe. Eso es un fallo valido.

## Green: escribir el codigo minimo para que pase

El objetivo de la fase Green es hacer que el test pase con la menor cantidad de codigo posible. No se busca la solucion correcta, ni elegante, ni completa. Se busca la solucion mas simple que satisfaga el test actual. Esta restriccion fuerza a pensar en lo estrictamente necesario y evita anticipar requisitos futuros.

```java
public class FizzBuzz {

    public String of(int number) {
        return "Fizz";
    }
}
```

El test pasa. La implementacion es trivial e incorrecta para cualquier otro valor, pero eso no importa ahora. El siguiente test obligara a generalizar.

## Refactor: mejorar sin cambiar comportamiento

Una vez que el test pasa, se dispone de una red de seguridad que permite modificar el codigo con confianza. En esta fase se eliminan duplicaciones, se mejoran nombres, se extraen metodos, se aplican patrones. El comportamiento debe permanecer identico; si un test falla durante el refactoring, el cambio introducido es incorrecto.

```java
public class FizzBuzz {

    public String of(int number) {
        if (number % 3 == 0) {
            return "Fizz";
        }
        // No hay else porque la funcion devolvera null para otros casos
        // El siguiente test obligara a manejarlo
        return null;
    }
}
```

## Repetir: anadir el siguiente comportamiento

Con el refactoring completado, se repite el ciclo para el siguiente comportamiento. Cada iteracion anade un caso nuevo y fuerza a generalizar la solucion.

```java
// Segundo test: Red
@Test
void shouldReturnBuzzWhenNumberIsDivisibleByFive() {
    FizzBuzz fizzbuzz = new FizzBuzz();
    assertEquals("Buzz", fizzbuzz.of(5));
}

// Segundo test: Green
public String of(int number) {
    if (number % 5 == 0) {
        return "Buzz";
    }
    if (number % 3 == 0) {
        return "Fizz";
    }
    return String.valueOf(number);
}
```

## Ejemplo completo paso a paso

Implementemos un calculador de descuentos. La regla de negocio es: si el total supera 100 EUR, se aplica un 10 % de descuento.

**Iteracion 1 - Red:**

```java
@Test
void shouldApplyTenPercentDiscountWhenTotalExceedsOneHundred() {
    DiscountCalculator calculator = new DiscountCalculator();
    assertEquals(90.0, calculator.calculate(100.0), 0.001);
}
```

**Iteracion 1 - Green:**

```java
public class DiscountCalculator {
    public double calculate(double total) {
        return total * 0.9;
    }
}
```

**Iteracion 1 - Refactor:** No hay nada que refactorizar.

**Iteracion 2 - Red:**

```java
@Test
void shouldNotApplyDiscountWhenTotalIsOneHundredOrLess() {
    DiscountCalculator calculator = new DiscountCalculator();
    assertEquals(80.0, calculator.calculate(80.0), 0.001);
}
```

**Iteracion 2 - Green:**

```java
public double calculate(double total) {
    if (total > 100.0) {
        return total * 0.9;
    }
    return total;
}
```

**Iteracion 2 - Refactor:**

```java
public double calculate(double total) {
    return total > 100.0 ? total * 0.9 : total;
}
```

> La clave del ciclo esta en la disciplina de no escribir codigo de produccion sin un test que falle previamente, y de no anadir mas codigo del necesario para que ese test pase.
