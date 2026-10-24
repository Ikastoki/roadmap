# Anotaciones basicas

JUnit 5 proporciona un conjunto de anotaciones que configuran el comportamiento de los tests, su identificacion, agrupacion y ejecucion. Estas anotaciones son la puerta de entrada al modelo de programacion de Jupiter y sustituyen a las convenciones de nombres y configuraciones XML de versiones anteriores.

## @Test

La anotacion fundamental que marca un metodo como un test. A diferencia de JUnit 4, @Test en JUnit 5 no tiene atributos (expected y timeout se han movido a aserciones). Cualquier metodo anotado con @Test debe ser public, no devolver valor (void) y no recibir parametros (salvo en tests parametrizados).

```java
@Test
void deberiaCalcularSuma() {
    var calculadora = new Calculadora();
    assertEquals(5, calculadora.sumar(2, 3));
}
```

## @DisplayName

Permite definir un nombre legible para la clase de test o el metodo, que se mostrara en los reportes y en el output del IDE. Soporta espacios, caracteres especiales y emojis (aunque estos ultimos no se recomiendan en entornos profesionales).

```java
@DisplayName("Calculo de descuentos para clientes premium")
@Test
void descuentoClientePremium() {
    // ...
}
```

## @Disabled

Sustituye a @Ignore de JUnit 4. Deshabilita un metodo o clase de test completo. Debe ir siempre acompanado de un comentario explicando el motivo, para que no se convierta en un test olvidado que nunca se reactiva.

```java
@Test
@Disabled("Pendiente de implementar tras refactorizacion del modulo de pagos")
void testPagoConTarjeta() {
    // ...
}
```

> Los tests anotados con @Disabled son deuda tecnica. Cada test deshabilitado reduce la cobertura efectiva y debe tener un ticket asociado para su reactivacion.

## @RepeatedTest

Ejecuta el mismo metodo de test un numero especificado de veces. Es util para pruebas de aleatoriedad, concurrencia o para verificar que un comportamiento es consistente en multiples ejecuciones. Cada repeticion puede tener un nombre personalizado mediante un patron.

```java
@RepeatedTest(value = 10, name = "Intento {currentRepetition} de {totalRepetitions}")
void testConexionBaseDatos() {
    assertTrue(servicio.estaDisponible());
}
```

## @TestFactory

Marca un metodo que produce una coleccion de DynamicTest (tests generados en tiempo de ejecucion). El metodo debe devolver un Stream, Collection, Iterable o array de DynamicTest. Se utiliza para generar tests basados en datos externos, configuraciones o combinaciones.

```java
@TestFactory
Stream<DynamicTest> testsParaOperacionesMatematicas() {
    return Stream.of(
        dynamicTest("Suma", () -> assertEquals(4, 2 + 2)),
        dynamicTest("Resta", () -> assertEquals(2, 4 - 2)),
        dynamicTest("Multiplicacion", () -> assertEquals(6, 2 * 3))
    );
}
```

## @Nested

Permite anidar clases de test dentro de otras clases de test, creando una jerarquia que se refleja en la ejecucion y en los reportes. Las clases anidadas deben ser clases internas estaticas (static inner classes) y pueden tener sus propios metodos de ciclo de vida (@BeforeEach, @AfterEach). Es especialmente util para organizar tests por funcionalidad o por estado del sistema bajo prueba.

```java
@DisplayName("Operaciones con usuario")
class UsuarioTest {

    @Nested
    @DisplayName("Cuando el usuario es nuevo")
    class UsuarioNuevoTest {
        @Test
        void deberiaTenerEstadoPendiente() { /* ... */ }
    }

    @Nested
    @DisplayName("Cuando el usuario esta activo")
    class UsuarioActivoTest {
        @Test
        void deberiaPermitirInicioSesion() { /* ... */ }
    }
}
```

## @Tag

Anade etiquetas a los tests para su filtrado selectivo durante la ejecucion. Las tags permiten crear categorias semanticas (rapido, lento, integracion, api, db) que se pueden incluir o excluir en la configuracion de build o en el lanzador. Las expresiones de tags soportan operadores logicos (AND, OR, NOT) para combinar filtros.

```java
@Tag("rapido")
@Tag("unitario")
@Test
void testValidacionEmail() { /* ... */ }

@Tag("lento")
@Tag("integracion")
@Test
void testConexionBaseDatos() { /* ... */ }
```

## @Order

En JUnit 5, el orden de ejecucion de los tests es indeterminado por defecto para fomentar la independencia entre tests. La anotacion @Order (junto con @TestMethodOrder(MethodName.class)) permite definir un orden explicito cuando sea estrictamente necesario. Debe usarse con prudencia, ya que ordenar los tests suele indicar dependencias ocultas entre ellos.

```java
@TestMethodOrder(OrderAnnotation.class)
class TestOrdenados {

    @Test
    @Order(1)
    void primerTest() { /* ... */ }

    @Test
    @Order(2)
    void segundoTest() { /* ... */ }
}
```
