# Excepciones custom

Las excepciones personalizadas (custom exceptions) son clases que extienden `Exception` o `RuntimeException` para modelar condiciones de error especificas del dominio. Verificar que estas excepciones transportan la informacion correcta es parte del testing de la capa de negocio.

## Testing de excepciones custom

Una excepcion custom tipica anade campos que contienen contexto sobre el error: codigo de error, identificador de la entidad afectada, valores que provocaron la condicion, etc.

```java
public class SaldoInsuficienteException extends RuntimeException {
    private final String cuentaId;
    private final BigDecimal saldoActual;
    private final BigDecimal montoRequerido;

    public SaldoInsuficienteException(String cuentaId, BigDecimal saldoActual, BigDecimal montoRequerido) {
        super(String.format("Cuenta %s: saldo %.2f insuficiente para %.2f", cuentaId, saldoActual, montoRequerido));
        this.cuentaId = cuentaId;
        this.saldoActual = saldoActual;
        this.montoRequerido = montoRequerido;
    }

    public String getCuentaId() { return cuentaId; }
    public BigDecimal getSaldoActual() { return saldoActual; }
    public BigDecimal getMontoRequerido() { return montoRequerido; }
}
```

El test debe verificar tanto el tipo de la excepcion como los valores de sus campos.

```java
@Test
void retirarConSaldoInsuficiente() {
    Cuenta cuenta = new Cuenta("C-001", new BigDecimal("100"));

    SaldoInsuficienteException ex = assertThrows(
        SaldoInsuficienteException.class,
        () -> cuenta.retirar(new BigDecimal("200"))
    );

    assertThat(ex.getCuentaId()).isEqualTo("C-001");
    assertThat(ex.getSaldoActual()).isEqualByComparingTo(new BigDecimal("100"));
    assertThat(ex.getMontoRequerido()).isEqualByComparingTo(new BigDecimal("200"));
}
```

## Verificar campos

Con AssertJ se pueden extraer campos directamente, evitando invocar getters. La combinacion de `extracting` y `tuple` permite verificar multiples campos en una sola asercion.

```java
@Test
void excepcionConCamposCustom() {
    assertThatThrownBy(() -> cuenta.retirar(new BigDecimal("200")))
        .isInstanceOf(SaldoInsuficienteException.class)
        .extracting("cuentaId", "saldoActual", "montoRequerido")
        .containsExactly("C-001", new BigDecimal("100"), new BigDecimal("200"));
}
```

Para campos con tipos complejos o cuando se necesita un matcher especifico:

```java
assertThatThrownBy(() -> cuenta.retirar(new BigDecimal("200")))
    .isInstanceOf(SaldoInsuficienteException.class)
    .satisfies(ex -> {
        assertThat(ex.getSaldoActual()).isGreaterThan(BigDecimal.ZERO);
        assertThat(ex.getMontoRequerido()).isGreaterThan(ex.getSaldoActual());
    });
```

## Stack trace

El stack trace no debe verificarse en los tests por las mismas razones que con las excepciones estandar: dependencia del entorno y fragilidad. Sin embargo, hay una excepcion: cuando se construyen excepciones anidadas y se quiere verificar que la causa se ha establecido correctamente.

```java
@Test
void excepcionConCausa() {
    IOException causa = new IOException("Error de red");

    ProcesamientoException ex = assertThrows(
        ProcesamientoException.class,
        () -> servicio.procesarConCausa(causa)
    );

    assertThat(ex).hasCause(causa);
    assertThat(ex.getCause()).hasMessage("Error de red");
}
```

## Testing de jerarquias

Si las excepciones custom forman una jerarquia, los tests deben verificar el tipo mas concreto, no la clase base.

```java
// Incorrecto
assertThrows(PagoException.class, () -> procesador.cobrar(null));

// Correcto
assertThrows(MontoInvalidoException.class, () -> procesador.cobrar(null));
```

La verificacion del tipo concreto garantiza que el subsistema de errores esta correctamente disenado y que los mecanismos de captura selectiva funcionan.
