# Parametrized tests

Los tests parametrizados permiten ejecutar un mismo metodo de test con multiples conjuntos de argumentos, eliminando la duplicacion de codigo cuando se necesita verificar el mismo comportamiento con diferentes entradas. JUnit 5 incorpora soporte nativo para tests parametrizados a traves de la anotacion @ParameterizedTest y diversas fuentes de argumentos.

## @ParameterizedTest

Sustituye a @Test para marcar un metodo como test parametrizado. El metodo debe declarar parametros que se inyectaran en cada ejecucion. Es necesario incluir al menos una fuente de argumentos (anotada adicionalmente) que proporcione los valores para esos parametros.

```java
@ParameterizedTest
@ValueSource(strings = { "user@example.com", "admin@test.org", "soporte@empresa.com" })
void deberiaAceptarEmailValido(String email) {
    assertTrue(ValidadorEmail.esValido(email));
}
```

La dependencia `junit-jupiter-params` debe estar incluida en el proyecto para habilitar esta funcionalidad.

## @ValueSource

Proporciona un array de valores literales del mismo tipo (String, int, long, double, boolean, char, byte, short, float). Es la fuente mas simple y se usa cuando se necesita probar con una lista plana de valores sin estructura adicional.

```java
@ParameterizedTest
@ValueSource(ints = { 1, 2, 4, 8, 16, 32 })
void deberiaSerPotenciaDeDos(int valor) {
    assertTrue(valor > 0 && (valor & (valor - 1)) == 0);
}
```

## @CsvSource

Proporciona argumentos en formato CSV (valores separados por comas). Es la fuente mas versatil para tests con multiples parametros, ya que permite definir varias columnas de datos en una sola linea. Los valores nulos se representan con `null` literal y los valores vacios con comillas simples.

```java
@ParameterizedTest
@CsvSource({
    "2, 3, 5",
    "10, 20, 30",
    "0, 0, 0",
    "-1, 1, 0"
})
void deberiaSumarCorrectamente(int a, int b, int esperado) {
    assertEquals(esperado, calculadora.sumar(a, b));
}
```

## @CsvFileSource

Similar a @CsvSource pero lee los datos de un archivo CSV externo. Es util cuando el conjunto de datos es grande y no es practico incluirlo en el codigo. El archivo se busca en el classpath y puede tener encabezados opcionales que se ignoran.

```java
@ParameterizedTest
@CsvFileSource(resources = "/datos/calculadora.csv", numLinesToSkip = 1)
void deberiaSumarDesdeArchivo(int a, int b, int esperado) {
    assertEquals(esperado, calculadora.sumar(a, b));
}
```

Contenido del archivo `calculadora.csv`:
```
a,b,esperado
2,3,5
10,20,30
0,0,0
```

## @MethodSource

Proporciona argumentos desde un metodo estatico que devuelve un Stream, Collection, Iterable o array. Es la fuente mas potente y flexible, ya que permite generar datos programaticamente, combinarlos desde multiples fuentes o leerlos de bases de datos y APIs. El metodo debe ser estatico y no recibir parametros.

```java
@ParameterizedTest
@MethodSource("proveedorUsuarios")
void deberiaValidarUsuarios(Usuario usuario, boolean esperado) {
    assertEquals(esperado, validador.usuarioValido(usuario));
}

static Stream<Arguments> proveedorUsuarios() {
    return Stream.of(
        Arguments.of(new Usuario("ana", "ana@email.com"), true),
        Arguments.of(new Usuario("", "invalido"), false),
        Arguments.of(new Usuario("luis", null), false)
    );
}
```

## @EnumSource

Proporciona valores desde un enum. Puede incluir todos los valores del enum o filtrar un subconjunto especifico. Es ideal para probar logica que se comporta diferente segun un tipo enumerado.

```java
@ParameterizedTest
@EnumSource(value = TipoCliente.class, names = { "VIP", "PREMIUM" })
void soloVipYPremiumTienenDescuento(TipoCliente tipo) {
    assertTrue(descuentoService.tieneDescuento(tipo));
}
```

## @NullSource y @EmptySource

@NullSource proporciona un valor null para el primer parametro, mientras que @EmptySource proporciona un valor vacio (cadena vacia, lista vacia, etc.). La combinacion @NullAndEmptySource proporciona ambos en una sola anotacion. Son especialmente utiles para probar la robustez del codigo frente a entradas nulas o vacias.

```java
@ParameterizedTest
@ValueSource(strings = { "texto", "otro" })
@NullSource
@EmptySource
void deberiaManejarEntradasExtremas(String entrada) {
    assertDoesNotThrow(() -> servicio.procesar(entrada));
}
```

## Personalizacion del nombre con {index} y {displayName}

Los tests parametrizados generan nombres automaticamente para cada invocacion. Se puede personalizar el patron usando los marcadores `{index}` (indice de la invocacion), `{displayName}` (nombre del metodo), `{arguments}` (argumentos completos) o `{0}`, `{1}`, etc. (argumentos individuales).

```java
@DisplayName("Suma de numeros")
@ParameterizedTest(name = "{displayName}: {0} + {1} = {2}")
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "10, -5, 5"
})
void testSuma(int a, int b, int esperado) {
    assertEquals(esperado, a + b);
}
```

La salida en el reporte mostrara nombres como "Suma de numeros: 1 + 1 = 2" en lugar del nombre generico del metodo, mejorando significativamente la legibilidad de los informes de ejecucion.
