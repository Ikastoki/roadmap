# Testing de excepciones

Verificar que el codigo lanza las excepciones correctas en las situaciones esperadas es una parte fundamental del testing defensivo. No basta con que el metodo no falle; debe fallar de forma controlada cuando las condiciones lo requieren.

## assertThrows JUnit 5

JUnit 5 introdujo `assertThrows`, un metodo estatico que ejecuta el codigo proporcionado y verifica que lanza una excepcion del tipo esperado. Devuelve la excepcion capturada para que se pueda inspeccionar.

```java
import static org.junit.jupiter.api.Assertions.assertThrows;

@Test
void lanzaExcepcionCuandoSaldoInsuficiente() {
    Cuenta cuenta = new Cuenta(100);

    SaldoInsuficienteException ex = assertThrows(
        SaldoInsuficienteException.class,
        () -> cuenta.retirar(200)
    );

    assertThat(ex.getMessage()).contains("Saldo insuficiente");
    assertThat(ex.getSaldoActual()).isEqualTo(100);
    assertThat(ex.getMontoRequerido()).isEqualTo(200);
}
```

La ventaja de `assertThrows` es que el test falla si el codigo no lanza la excepcion o si lanza una de tipo distinto. Ademas, el test sigue ejecutandose si la excepcion se captura correctamente, permitiendo verificar propiedades del objeto excepcion.

## assertThatThrownBy AssertJ

AssertJ ofrece una alternativa mas expresiva con su API fluida.

```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@Test
void lanzaExcepcionConMensaje() {
    assertThatThrownBy(() -> new Cuenta(-50))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("El saldo inicial no puede ser negativo")
        .hasNoCause();
}
```

AssertJ permite encadenar verificaciones sobre la excepcion: tipo, mensaje, causa, mensaje de causa, y propiedades especificas de la excepcion.

```java
assertThatThrownBy(() -> servicio.procesar(null))
    .isInstanceOf(ValidationException.class)
    .hasMessageContaining("El parametro no puede ser nulo")
    .extracting("codigoError", as(InstanceOfAssertFactories.INT))
    .isEqualTo(400);
```

## Verificar tipo, mensaje y causa

Una buena practica es verificar los tres niveles de informacion de una excepcion:

1. **Tipo**: que la excepcion sea de la clase esperada (no cualquier `RuntimeException`).
2. **Mensaje**: que contenga la informacion descriptiva adecuada.
3. **Causa**: si la excepcion envuelve otra, verificar la causa raiz.

```java
@Test
void excepcionEnvuelveCausa() {
    ServicioExterno servicio = mock(ServicioExterno.class);
    when(servicio.llamar()).thenThrow(new IOException("Timeout"));

    assertThatThrownBy(() -> fachada.ejecutar(servicio))
        .isInstanceOf(ProcesamientoException.class)
        .hasMessage("Error al procesar la solicitud")
        .hasCauseInstanceOf(IOException.class)
        .hasRootCauseMessage("Timeout");
}
```

## No verificar el stack trace

El stack trace no debe verificarse en tests porque depende del entorno de ejecucion, la profundidad de la pila y las optimizaciones del JIT. Verificar lineas especificas del stack trace produce tests fragiles que fallan sin motivo real.

> `assertThrows` y `assertThatThrownBy` solo verifican que se lanza una excepcion. Si el metodo bajo prueba captura la excepcion internamente y no la relanza, el test no la detectara. En esos casos el test debe verificar el estado posterior del objeto o el comportamiento alternativo.
