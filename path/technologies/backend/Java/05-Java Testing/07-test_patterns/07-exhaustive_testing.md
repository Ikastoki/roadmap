# Exhaustive Testing

Probar todos los casos posibles de un comportamiento es, en la mayoria de los sistemas, imposible. El espacio de entradas de cualquier funcion con parametros numericos o de cadena es infinito o practicamente infinito. Sin embargo, para ciertos dominios acotados, el testing exhaustivo es viable y recomendable.

## Probar todos los casos

El testing exhaustivo consiste en verificar que la salida es correcta para cada entrada posible dentro de un dominio finito. Es aplicable cuando:

- El dominio de entrada es pequeno y conocido
- El numero de combinaciones es manejable
- El coste de ejecutar todos los casos es aceptable
- La correccion del comportamiento es critica

```java
class CardSuitTest {

    @ParameterizedTest
    @EnumSource(CardSuit.class)
    void shouldRecognizeAllSuits(CardSuit suit) {
        assertTrue(CardValidator.isValidSuit(suit));
    }
}
```

En este ejemplo, el enum `CardSuit` tiene 4 valores. Probar los 4 casos no solo es posible, sino trivial.

## Parametrized tests

JUnit 5 proporciona `@ParameterizedTest` para ejecutar el mismo test con multiples entradas. Combinado con `@MethodSource`, `@CsvSource`, `@ValueSource` o `@EnumSource`, permite cubrir multiples casos sin duplicar codigo.

```java
class PasswordValidatorTest {

    @ParameterizedTest
    @ValueSource(strings = {
        "abc123",          // valida
        "pass1234",        // valida
        "secureP@ss1",     // valida
        "Abcd1234"         // valida
    })
    void shouldAcceptValidPasswords(String password) {
        assertTrue(PasswordValidator.isValid(password));
    }

    @ParameterizedTest
    @ValueSource(strings = {
        "abc",             // demasiado corta
        "12345678",        // solo numeros
        "abcdefgh",        // solo letras
        "",                // vacia
        "1234 5678"        // con espacios
    })
    void shouldRejectInvalidPasswords(String password) {
        assertFalse(PasswordValidator.isValid(password));
    }
}
```

## Cuando es viable

El testing exhaustivo es viable cuando se cumple al menos una de estas condiciones:

- **Dominio enumerable pequeno**: enums, booleanos, rangos de enteros pequenos
- **Combinaciones acotadas**: todas las permutaciones de un conjunto fijo de opciones
- **Propiedades matematicas**: funciones donde se puede demostrar que probar un subconjunto cubre todos los casos (por ejemplo, funciones lineales)
- **Tests de aceptacion**: se prueban todas las rutas de una funcionalidad critica

```java
@Test
void shouldHandleAllOrderStates() {
    for (OrderState state : OrderState.values()) {
        Order order = new Order(state);
        assertNotNull(order.statusDescription());
    }
}
```

## Example-based vs Exhaustive

| Aspecto | Example-based | Exhaustive |
|---------|---------------|------------|
| Cobertura | Subconjunto representativo | Totalidad del dominio |
| Coste de creacion | Bajo | Medio (puede requerir generacion) |
| Coste de ejecucion | Bajo | Puede ser alto |
| Confianza | Limitada a los ejemplos | Maxima dentro del dominio |
| Mantenimiento | Bajo | Puede requerir actualizacion cuando el dominio cambia |

En la practica, la mayoria de los tests son example-based. El testing exhaustivo se reserva para dominios pequenos o logicas de negocio criticas donde un error tendria consecuencias graves.

## Combinacion de enfoques

La estrategia mas efectiva combina ambos enfoques: tests exhaustivos para dominios pequenos y example-based para el resto, complementados con property-based testing para propiedades invariantes.

```java
class RomanNumeralConverterTest {

    // Exhaustivo para valores pequenos
    @ParameterizedTest
    @CsvSource({
        "1, I",
        "2, II",
        "3, III",
        "4, IV",
        "5, V",
        "6, VI",
        "7, VII",
        "8, VIII",
        "9, IX",
        "10, X"
    })
    void shouldConvertSmallNumbers(int number, String expected) {
        assertEquals(expected, RomanNumeralConverter.convert(number));
    }

    // Example-based para valores grandes
    @Test
    void shouldConvertFifty() {
        assertEquals("L", RomanNumeralConverter.convert(50));
    }

    @Test
    void shouldConvertNinetyNine() {
        assertEquals("XCIX", RomanNumeralConverter.convert(99));
    }

    @Test
    void shouldConvertThousand() {
        assertEquals("M", RomanNumeralConverter.convert(1000));
    }
}
```

> La cobertura exhaustiva sobre un subconjunto pequeno del dominio combinada con ejemplos representativos del resto ofrece un balance optimo entre confianza y coste de mantenimiento.

Cuando el dominio es demasiado grande para ser exhaustivo pero las consecuencias de un error son graves, el property-based testing (con herramientas como jqwik) puede generar cientos o miles de casos aleatorios, acercandose al ideal exhaustivo sin requerir la enumeracion manual de todas las entradas.
