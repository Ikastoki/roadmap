# Tests dinamicos

Los tests dinamicos permiten generar casos de prueba en tiempo de ejecucion, dentro de un metodo de fabrica anotado con @TestFactory. A diferencia de los tests estaticos (definidos con @Test en tiempo de compilacion), los tests dinamicos se construyen programaticamente, lo que abre la puerta a escenarios donde los casos de prueba no se conocen hasta el momento de la ejecucion.

## @TestFactory

La anotacion @TestFactory marca un metodo como una fabrica de tests dinamicos. El metodo no es un test en si mismo, sino que devuelve una coleccion de instancias DynamicTest. El metodo debe devolver Stream, Collection, Iterable, Iterator o array de DynamicTest. A diferencia de los metodos @Test, los metodos @TestFactory no pueden ser privados ni estaticos (salvo con Lifecycle.PER_CLASS).

```java
@TestFactory
Stream<DynamicTest> testsDinamicosParaValidacion() {
    return Stream.of(
        dynamicTest("Email valido", () -> assertTrue(Validador.emailEsValido("a@b.com"))),
        dynamicTest("Email sin arroba", () -> assertFalse(Validador.emailEsValido("invalido"))),
        dynamicTest("Email nulo", () -> assertThrows(NullPointerException.class,
            () -> Validador.emailEsValido(null)))
    );
}
```

## DynamicTest

Cada DynamicTest se construye con dos elementos: un nombre (String o DynamicTest.stream con DisplayNameGenerator) y un bloque de codigo ejecutable (Implementacion de la interfaz funcional Executable). El nombre se mostrara en los reportes de ejecucion como si fuera un @DisplayName de un test estatico.

```java
@TestFactory
Stream<DynamicTest> testsDesdeArchivo() throws IOException {
    var lineas = Files.readAllLines(Paths.get("casos_test.csv"));
    return lineas.stream().map(linea -> {
        var partes = linea.split(",");
        var entrada = Integer.parseInt(partes[0]);
        var esperado = Integer.parseInt(partes[1]);
        return DynamicTest.dynamicTest(
            "Test con entrada " + entrada,
            () -> assertEquals(esperado, servicio.procesar(entrada))
        );
    });
}
```

El bloque Executable puede lanzar cualquier excepcion. Si lanza una excepcion, el DynamicTest se marca como fallido. Si el metodo @TestFactory lanza una excepcion durante la generacion de los tests, toda la fabrica se marca como fallida y ningun DynamicTest se ejecuta.

## DynamicContainer

Los DynamicContainer permiten agrupar DynamicTest en una estructura jerarquica, similar a como @Nested agrupa clases de test estaticas. Cada DynamicContainer tiene un nombre y una coleccion de DynamicTest o DynamicContainer hijos, creando un arbol que se refleja en la salida de ejecucion.

```java
@TestFactory
DynamicNode testsAgrupadosPorCategoria() {
    return DynamicContainer.dynamicContainer("Validaciones",
        Stream.of(
            DynamicContainer.dynamicContainer("Email",
                Stream.of(
                    DynamicTest.dynamicTest("Email valido",
                        () -> assertTrue(Validador.emailEsValido("a@b.com"))),
                    DynamicTest.dynamicTest("Email invalido",
                        () -> assertFalse(Validador.emailEsValido("invalido")))
                )),
            DynamicContainer.dynamicContainer("Telefono",
                Stream.of(
                    DynamicTest.dynamicTest("Telefono valido",
                        () -> assertTrue(Validador.telefonoEsValido("+34911111111"))),
                    DynamicTest.dynamicTest("Telefono corto",
                        () -> assertFalse(Validador.telefonoEsValido("123")))
                ))
        ));
}
```

> DynamicContainer ofrece una alternativa programatica a @Nested para organizar tests. Sin embargo, la estructura no se refleja en anotaciones de ciclo de vida: no existen @BeforeEach ni @AfterEach para un contenedor especifico.

## Generacion en runtime

La principal ventaja de los tests dinamicos es la capacidad de generar casos de prueba basados en datos que solo estan disponibles en tiempo de ejecucion. Algunos escenarios donde esta capacidad resulta especialmente valiosa incluyen:

- **Datos de archivos externos:** leer casos de prueba desde CSV, JSON, XML o bases de datos.
- **Exploracion de APIs:** generar tests automaticamente a partir del esquema de una API REST o GraphQL.
- **Cobertura de combinaciones:** generar todas las combinaciones posibles de parametros para pruebas de interfaz.
- **Tests basados en metadatos:** leer anotaciones, reflexion o archivos de configuracion para determinar que probar.
- **Pruebas de mutacion:** ejecutar tests con diferentes mutaciones del codigo bajo prueba.

```java
@TestFactory
Stream<DynamicTest> testsDesdeConfiguracion() {
    var config = CargadorConfiguracion.cargar("reglas-validacion.json");
    return config.getReglas().stream()
        .map(regla -> DynamicTest.dynamicTest(
            "Regla: " + regla.getDescripcion(),
            () -> {
                var resultado = validador.aplicarRegla(regla);
                assertTrue(resultado.esValida(), resultado.getMensajeError());
            }
        ));
}
```

A pesar de su flexibilidad, los tests dinamicos tienen limitaciones importantes. No soportan callbacks de ciclo de vida (@BeforeEach, @AfterEach) por test individual, el orden de ejecucion no es controlable mediante @Order, y las herramientas de reporte y los IDEs pueden mostrar la estructura dinamica de forma menos clara que los tests estaticos. Por estas razones, los tests dinamicos se recomiendan como complemento, no como sustituto, de los tests basados en @Test.
