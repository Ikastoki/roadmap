# Naming conventions

El nombre de un test es su primera forma de documentacion. Un test bien nombrado comunica que comportamiento verifica, en que condiciones y cual es el resultado esperado, sin necesidad de leer el cuerpo del metodo.

## should_behavior_when_condition

El patron `should_behavior_when_condition` es el mas adoptado en la comunidad Java. Sigue la estructura: "deberia [comportamiento] cuando [condicion]".

```java
@Test
void should_return_user_when_valid_email() {
    // ...
}

@Test
void should_throw_exception_when_email_already_exists() {
    // ...
}

@Test
void should_calculate_discount_when_vip_customer() {
    // ...
}
```

Este patron es legible, descriptivo y funciona igualmente bien en espanol e ingles. La eleccion del idioma debe ser consistente en todo el proyecto.

## ClaseNombreTest

La clase de test se nombra con el nombre de la clase bajo prueba seguido del sufijo `Test`.

```java
class UsuarioServiceTest { }
class FacturaRepositoryTest { }
class EmailValidatorTest { }
```

Para tests de integracion se usa el sufijo `IT` o `IntegrationTest`, y el plugin Failsafe se configura para incluirlos.

```java
class UsuarioServiceIT { }
class FacturaRepositoryIntegrationTest { }
```

## @DisplayName

Ademas del nombre del metodo, JUnit 5 permite anadir `@DisplayName` para descripciones mas extensas o con formato legible.

```java
@DisplayName("Servicio de autenticacion")
class LoginServiceTest {

    @Test
    @DisplayName("Debe rechazar credenciales cuando la contrasena expiro")
    void should_reject_when_password_expired() {
        // ...
    }
}
```

Los `@DisplayName` aparecen en los reportes de ejecucion y en los IDEs, lo que facilita la revision de resultados sin necesidad de leer el codigo.

## Patrones alternativos

Existen otras convenciones validas que algunos equipos prefieren:

- `methodName_expectedBehavior()`: `calcularDescuento_shouldReturnZeroWhenNegativeAmount()`.
- `test` + descripcion: `testDescuentoVip` (estilo antiguo, no recomendado).
- BDD con `given_when_then`: `givenVipCustomer_whenCalculatingDiscount_thenApply20Percent()`.

El patron BDD (`given_when_then`) es especialmente util cuando el test tiene multiples pasos y se quiere reflejar la estructura Given-When-Then directamente en el nombre.

```java
@Test
void givenVipCustomer_whenCalculatingDiscount_thenApply20Percent() {
    // given
    Usuario vip = new Usuario(TipoCliente.VIP);
    // when
    double descuento = calculador.calcular(vip, 100.0);
    // then
    assertThat(descuento).isEqualTo(20.0);
}
```

## Reglas de oro

Independientemente del patron elegido, las reglas fundamentales son:

- El nombre debe ser legible sin necesidad de leer el codigo del test.
- Debe incluir la condicion y el resultado esperado.
- Debe ser consistente en todo el proyecto (mismo patron, mismo idioma).
- Si el nombre necesita mas de 80 caracteres, probablemente el test hace demasiadas cosas y deberia dividirse.
- Los metodos de test no llevan la anotacion `@DisplayName` y el nombre del metodo se duplican. Se usa uno u otro, no ambos.

La convencion de nomenclatura se acuerda en equipo y se documenta. No hay un unico patron correcto, pero la consistencia es obligatoria. La ausencia de convencion genera tests cuyo proposito es ambiguo y que con el tiempo nadie se atreve a modificar.
