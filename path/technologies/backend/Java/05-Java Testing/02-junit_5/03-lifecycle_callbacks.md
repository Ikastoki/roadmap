# Lifecycle callbacks

JUnit 5 proporciona anotaciones de ciclo de vida que permiten ejecutar codigo antes o despues de los tests a nivel de metodo o de clase. Estos callbacks son esenciales para inicializar recursos, configurar entornos de prueba y limpiar el estado entre ejecuciones, garantizando que cada test comience en un estado conocido y reproducible.

## @BeforeAll

El metodo anotado con @BeforeAll se ejecuta una unica vez antes de todos los tests de la clase. Debe ser un metodo estatico (a menos que se configure el ciclo de vida por instancia con @TestInstance(Lifecycle.PER_CLASS)). Es el lugar adecuado para inicializar recursos costosos que se reutilizan en todos los tests: conexiones a bases de datos, contenedores Docker, configuracion de sistemas externos o carga de datos maestros.

```java
@BeforeAll
static void inicializarBaseDeDatos() {
    var datasource = DatabaseManager.createInMemory();
    DatabaseManager.ejecutarScript("schema.sql");
}
```

## @BeforeEach

Se ejecuta antes de cada metodo @Test, @RepeatedTest, @ParameterizedTest o @TestFactory. Es el punto optimo para preparar el estado necesario para cada test individual: crear instancias del sistema bajo prueba, inicializar mocks, establecer datos de prueba especificos o limpiar estado compartido que podria contaminar la ejecucion.

```java
@BeforeEach
void setUp() {
    repositorio = new RepositorioUsuarioFake();
    servicio = new ServicioUsuario(repositorio);
    mockEmail = mock(ServicioEmail.class);
}
```

> @BeforeEach garantiza aislamiento: cada test parte de un estado fresco. Si los tests comparten estado mutable sin reinicializarlo, se introducen dependencias ocultas que provocan fallos intermitentes.

## @AfterEach

Se ejecuta despues de cada test, independientemente de si el test pasa, falla o lanza una excepcion. Es el contrapunto de @BeforeEach y se utiliza para liberar recursos que no deben persistir entre tests: cerrar archivos temporales, eliminar registros de base de datos, resetear contadores o verificar que no quedaron efectos secundarios.

```java
@AfterEach
void limpiar() {
    repositorio.limpiarTodos();
    mockEmail.resetear();
}
```

## @AfterAll

Se ejecuta una unica vez despues de todos los tests de la clase. Debe ser estatico (salvo con Lifecycle.PER_CLASS). Aqui se liberan los recursos costosos inicializados en @BeforeAll: cerrar conexiones, detener contenedores, eliminar directorios temporales o generar reportes de cobertura.

```java
@AfterAll
static void cerrarRecursos() {
    DatabaseManager.cerrarPool();
}
```

## @TempDir

JUnit 5 introduce la inyeccion de parametros para metodos de test y de ciclo de vida. @TempDir crea un directorio temporal unico para cada test (o clase) y se lo inyecta al metodo como parametro de tipo Path o File. El directorio se crea antes de la ejecucion y se elimina automaticamente al finalizar. Es especialmente util para tests que trabajan con el sistema de ficheros: procesamiento de archivos, generacion de reportes, exportaciones e importaciones.

```java
@Test
void deberiaEscribirArchivoDeReporte(@TempDir Path directorioTemporal) {
    var archivo = directorioTemporal.resolve("reporte.csv");
    servicio.generarReporte(archivo.toString());

    assertTrue(Files.exists(archivo));
    assertTrue(Files.size(archivo) > 0);
}
```

@TempDir tambien puede usarse como anotacion en un campo de instancia o estatico dentro de la clase de test, para compartir el directorio entre varios metodos:

```java
class TestConDirectorioTemporal {

    @TempDir
    Path directorioCompartido;

    @Test
    void testUno() {
        var archivo = directorioCompartido.resolve("datos.txt");
        // ...
    }

    @Test
    void testDos() {
        var archivo = directorioCompartido.resolve("datos.txt");
        // Mismo directorio, mismo estado
    }
}
```

> La limpieza automatica de @TempDir evita la acumulacion de archivos residuales en el sistema. Sin embargo, si el test falla, JUnit 5 no elimina el directorio para permitir el analisis forense del fallo.
