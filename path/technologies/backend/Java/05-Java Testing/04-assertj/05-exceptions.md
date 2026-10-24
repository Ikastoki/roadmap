# Exceptions

La verificacion de excepciones es uno de los puntos donde AssertJ marca una diferencia notable respecto a las aproximaciones clasicas. Mientras que JUnit 5 ofrece `assertThrows` para capturar excepciones, AssertJ proporciona una API fluida que permite no solo verificar el tipo de excepcion lanzada, sino tambien inspeccionar su mensaje, causa, stack trace y propiedades personalizadas en una sola cadena de llamadas.

## assertThatThrownBy

`assertThatThrownBy` es el metodo principal para capturar y verificar excepciones. Recibe un bloque lambda `ThrowableRunning` que debe contener el codigo que se espera que lance una excepcion. Retorna un `AbstractThrowableAssert` sobre el que se encadenan las verificaciones.

```java
assertThatThrownBy(() -> {
        int resultado = 1 / 0;
    })
    .isInstanceOf(ArithmeticException.class)
    .hasMessage("/ by zero");
```

A diferencia de `assertThrows` de JUnit, no es necesario asignar la excepcion a una variable para inspeccionarla; toda la verificacion ocurre en la cadena fluida.

## hasMessage / hasMessageContaining

`hasMessage` verifica que el mensaje de la excepcion coincida exactamente con el string esperado. `hasMessageContaining` solo requiere que el mensaje contenga una subcadena, util cuando el mensaje incluye informacion variable como IDs o timestamps.

```java
assertThatThrownBy(() -> {
        throw new IllegalArgumentException("El campo email es obligatorio");
    })
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("El campo email es obligatorio")
    .hasMessageContaining("email");
```

Tambien existen `hasMessageStartingWith`, `hasMessageEndingWith` y `hasMessageMatching` (que acepta una expresion regular), cubriendo practicamente cualquier escenario de validacion de mensajes.

## hasCause / hasRootCauseInstanceOf

`hasCause` verifica que la causa directa de la excepcion (obtenida via `getCause()`) sea igual a la proporcionada. `hasCauseInstanceOf` verifica solo el tipo de la causa. `hasRootCauseInstanceOf` navega hasta la causa raiz de toda la cadena de excepciones.

```java
Throwable causa = new IllegalArgumentException("causa raiz");
Throwable excepcion = new RuntimeException("error", causa);

assertThatThrownBy(() -> { throw excepcion; })
    .isInstanceOf(RuntimeException.class)
    .hasCause(causa)
    .hasCauseInstanceOf(IllegalArgumentException.class)
    .hasRootCauseInstanceOf(IllegalArgumentException.class);
```

> Cuando una excepcion envuelve a otra (patron _Exception Chaining_), `hasCause` verifica la causa inmediata mientras que `hasRootCause` recorre toda la cadena hasta la excepcion fundacional. Para cadenas profundas, usar `hasRootCause` evita tener que inspeccionar cada nivel intermedio.

## assertThatCode / doesNotThrowAnyException

`assertThatCode` ejecuta un bloque de codigo sin esperar que lance una excepcion. Se utiliza para verificar que un metodo no lanza excepciones bajo condiciones especificas. El metodo `doesNotThrowAnyException` confirma que el bloque completo se ejecuto sin lanzar ninguna excepcion comprobada ni no comprobada.

```java
assertThatCode(() -> {
        metodoQueNoDebeFallar();
    })
    .doesNotThrowAnyException();
```

Si el bloque lanza alguna excepcion, la asercion falla mostrando el tipo y mensaje de la excepcion no esperada. No debe confundirse con un try-catch silencioso; si la excepcion ocurre, el test falla con informacion diagnostica completa.

## assertThatExceptionOfType

`assertThatExceptionOfType` permite especificar de antemano el tipo de excepcion esperada, proporcionando un asertor tipado directamente. Es util en scenarios donde se quiere aprovechar el tipado fuerte para acceder a metodos especificos de la subclase de excepcion.

```java
assertThatExceptionOfType(IllegalArgumentException.class)
    .isThrownBy(() -> { metodoQueLanzaExcepcion(); })
    .withMessageContaining("invalido");
```

> La diferencia principal con `assertThatThrownBy` es que `assertThatExceptionOfType` exige declarar el tipo de excepcion en el nombre del metodo, haciendo la intencion mas explicita. En caso de que no se lance ninguna excepcion del tipo declarado, la asercion falla con un mensaje claro, a diferencia de un `assertThrows` que devolveria `null` y provocaria un `NullPointerException` en cascada.

```java
// Comparacion con JUnit 5
assertThrows(IllegalArgumentException.class, () -> metodo());

// AssertJ equivalente
assertThatThrownBy(() -> metodo())
    .isInstanceOf(IllegalArgumentException.class);
```

La API de excepciones de AssertJ incluye ademas `catchThrowable`, una alternativa funcional que captura cualquier `Throwable` lanzado por un bloque y lo retorna para su inspeccion posterior, permitiendo separar la captura de la verificacion:

```java
Throwable lanzado = catchThrowable(() -> metodo());
assertThat(lanzado).isInstanceOf(IllegalArgumentException.class);
```
