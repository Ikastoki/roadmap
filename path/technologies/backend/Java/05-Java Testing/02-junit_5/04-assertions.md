# Assertions

Las afirmaciones (assertions) son el mecanismo mediante el cual un test verifica que el comportamiento observado coincide con el esperado. JUnit 5 proporciona un conjunto completo de metodos estaticos en la clase `org.junit.jupiter.api.Assertions` para cubrir desde comparaciones simples hasta verificaciones complejas de excepciones, tiempos de ejecucion y agrupaciones.

## assertEquals y assertNotEquals

Verifican que dos valores son iguales o diferentes respectivamente. Utilizan el metodo `equals()` del objeto para la comparacion, y en el caso de tipos primitivos, la comparacion directa. Es recomendable incluir siempre un mensaje descriptivo como tercer argumento para facilitar el diagnostico de fallos.

```java
assertEquals(4, calculadora.sumar(2, 2), "La suma de 2+2 debe ser 4");
assertNotEquals(5, calculadora.sumar(2, 2), "La suma de 2+2 no debe ser 5");
```

Para comparaciones de punto flotante se debe proporcionar un delta (tolerancia) debido a la imprecision inherente de la aritmetica de coma flotante:

```java
assertEquals(3.1415, calculadora.pi(), 0.0001, "El valor de pi debe aproximarse a 3.1415");
```

## assertTrue y assertFalse

Verifican que una condicion booleana es verdadera o falsa. Son utiles para validar predicados, resultados de metodos que devuelven boolean o estados de objetos.

```java
assertTrue(usuario.estaActivo(), "El usuario deberia estar activo tras la confirmacion");
assertFalse(usuario.tieneBloqueo(), "El usuario no deberia tener bloqueo activo");
```

## assertNull y assertNotNull

Comprueban si un objeto es nulo o no nulo. Son especialmente importantes en APIs que pueden devolver null como resultado de busquedas o como indicador de ausencia.

```java
assertNotNull(servicio.buscarPorId(1), "El usuario con ID 1 deberia existir");
assertNull(servicio.buscarPorId(999), "El usuario con ID 999 no deberia existir");
```

## assertThrows

Verifica que la ejecucion de un bloque de codigo lanza una excepcion especifica. Es la sustitucion del atributo `expected` de JUnit 4 y proporciona la ventaja adicional de devolver la excepcion para inspeccionar sus propiedades (mensaje, causa, codigo de error).

```java
@Test
void deberiaLanzarExcepcionCuandoSaldoInsuficiente() {
    var cuenta = new CuentaBancaria(100.0);
    var excepcion = assertThrows(SaldoInsuficienteException.class,
        () -> cuenta.retirar(200.0));
    assertEquals("Saldo insuficiente: 100.0 disponibles", excepcion.getMessage());
}
```

> assertThrows solo verifica que se lanza la excepcion indicada o una subclase de la misma. Si se necesita una coincidencia exacta de tipo, debe anadirse una verificacion adicional con getClass().

## assertTimeout y assertTimeoutPreemptively

Verifican que un bloque de codigo se completa dentro de un tiempo maximo especificado. `assertTimeout` espera a que el bloque termine incluso si excede el tiempo (el test falla al finalizar), mientras que `assertTimeoutPreemptively` aborta el bloque si se supera el limite. La version preemptiva debe usarse con precaucion, ya que la interrupcion abrupta puede dejar recursos en estado inconsistente.

```java
assertTimeout(Duration.ofMillis(500),
    () -> servicio.procesarLote(datos),
    "El procesamiento del lote no debe exceder 500ms");

assertTimeoutPreemptively(Duration.ofSeconds(2),
    () -> servicio.externo.consultar(),
    "La consulta al servicio externo debe responder en menos de 2 segundos");
```

## assertAll

Agrupa multiples aserciones y las ejecuta todas, reportando cualquier fallo al final. Sin assertAll, la primera asercion que falla detiene la ejecucion del test, ocultando posibles fallos adicionales. assertAll es especialmente util para validar multiples propiedades de un objeto de forma exhaustiva en una sola ejecucion.

```java
@Test
void deberiaCrearUsuarioConDatosCorrectos() {
    var usuario = servicio.crearUsuario("ana@email.com", "Ana", "Garcia");
    assertAll("Validacion de usuario creado",
        () -> assertNotNull(usuario.getId()),
        () -> assertEquals("ana@email.com", usuario.getEmail()),
        () -> assertEquals("Ana", usuario.getNombre()),
        () -> assertEquals("Garcia", usuario.getApellido()),
        () -> assertTrue(usuario.estaActivo())
    );
}
```

## fail

Falla el test directamente con un mensaje opcional. Se utiliza en situaciones donde el flujo de ejecucion no deberia alcanzar un punto determinado: bloques catch que no deberian ejecutarse, ramas de codigo no soportadas o pruebas de funcionalidad no implementada.

```java
@Test
void metodoNoSoportado() {
    try {
        servicio.metodoExperimental();
        fail("Deberia lanzar UnsupportedOperationException");
    } catch (UnsupportedOperationException e) {
        // Excepcion esperada
    }
}
```

## assertDoesNotThrow

Verifica que un bloque de codigo se ejecuta sin lanzar ninguna excepcion. Es la contraparte de assertThrows y resulta util cuando una API checked exception obliga a capturar excepciones que no se esperan en el escenario de prueba.

```java
@Test
void deberiaProcesarSinErrores() {
    assertDoesNotThrow(() -> servicio.procesar(entradaValida),
        "El procesamiento de datos validos no debe lanzar excepcion");
}
```
