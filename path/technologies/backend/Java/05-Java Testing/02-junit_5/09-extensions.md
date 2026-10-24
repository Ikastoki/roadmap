# JUnit 5 extensions

El modelo de extensiones de JUnit 5 reemplaza el mecanismo de @RunWith y Rules de JUnit 4 con un sistema de interfaces de callback que permite interceptar y modificar el comportamiento del framework en puntos especificos del ciclo de vida de ejecucion. Este modelo es mas modular, componible y expresivo que el anterior.

## Extension Model

Una extension en JUnit 5 es cualquier clase que implementa una o mas interfaces del paquete `org.junit.jupiter.api.extension`. Cada interfaz define un punto de extension (callback) en el que la extension puede intervenir: antes de todos los tests, antes de cada test, cuando un test lanza una excepcion, despues de la ejecucion, etc. A diferencia de JUnit 4, multiples extensiones pueden coexistir y combinarse sin conflictos de herencia.

La extension se registra mediante la anotacion @ExtendWith a nivel de metodo, clase o mediante el mecanismo de registro automatico (ServiceLoader) para extensiones globales.

## @ExtendWith

Registra una o mas extensiones para la clase o metodo anotado. Las extensiones pueden ser clases que implementan interfaces de extension o anotaciones compuestas que actuan como meta-anotaciones.

```java
@ExtendWith({ MockitoExtension.class, BaseDeDatosExtension.class })
class UsuarioServiceTest {
    // Tests con soporte de Mockito y base de datos
}
```

## Interfaz BeforeAllCallback

Se ejecuta antes de que se invoque cualquier metodo @BeforeAll. Es util para inicializar recursos globales como contenedores Docker, pools de conexiones o configuracion de sistemas externos.

```java
public class ContenedorExtension implements BeforeAllCallback {

    @Override
    public void beforeAll(ExtensionContext context) {
        var contenedor = new GenericContainer<>("postgres:15")
            .withExposedPorts(5432);
        contenedor.start();
        // Almacenar referencia para limpieza posterior
        context.getStore(ExtensionContext.Namespace.GLOBAL)
            .put("contenedor", contenedor);
    }
}
```

## Interfaz AfterAllCallback

Se ejecuta despues de todos los tests y de los metodos @AfterAll. Es el contrapunto de BeforeAllCallback y se utiliza para liberar recursos globales.

```java
public class ContenedorExtension implements AfterAllCallback {

    @Override
    public void afterAll(ExtensionContext context) {
        var contenedor = context.getStore(ExtensionContext.Namespace.GLOBAL)
            .get("contenedor", GenericContainer.class);
        if (contenedor != null) {
            contenedor.stop();
        }
    }
}
```

## Interfaz BeforeEachCallback

Se ejecuta antes de cada metodo @Test, @RepeatedTest, @ParameterizedTest o @TestFactory. Equivale a anadir logica adicional a @BeforeEach pero de forma externa y reutilizable.

```java
public class TemporizadorExtension implements BeforeEachCallback {

    @Override
    public void beforeEach(ExtensionContext context) {
        var testName = context.getRequiredTestMethod().getName();
        System.out.println("Iniciando test: " + testName);
    }
}
```

## Interfaz AfterEachCallback

Se ejecuta despues de cada test, similar a @AfterEach pero como extension externa. Es util para tareas de limpieza o registro que deben ocurrir independientemente del resultado del test.

```java
public class CapturaResultadoExtension implements AfterEachCallback {

    @Override
    public void afterEach(ExtensionContext context) {
        var resultado = context.getExecutionException()
            .map(e -> "FALLIDO: " + e.getMessage())
            .orElse("EXITOSO");
        // Enviar resultado a sistema de reportes
    }
}
```

## Interfaz TestExecutionExceptionHandler

Permite manejar excepciones lanzadas durante la ejecucion de un test. Puede transformar la excepcion, registrar informacion adicional o incluso suprimir el fallo si la excepcion se considera aceptable bajo ciertas condiciones.

```java
public class IgnorarExcepcionRedExtension implements TestExecutionExceptionHandler {

    @Override
    public void handleTestExecutionException(ExtensionContext context,
                                              Throwable throwable) throws Throwable {
        if (throwable instanceof ConnectException) {
            // Registrar y suprimir: el servicio externo no esta disponible
            System.err.println("Servicio externo no disponible: " + throwable.getMessage());
            return; // El test continua sin fallar
        }
        throw throwable;
    }
}
```

> TestExecutionExceptionHandler debe usarse con extrema precaucion. Suprimir excepciones indiscriminadamente puede ocultar defectos reales y generar falsos positivos en la suite de tests.

## Crear una extension personalizada

Para crear una extension propia, se implementan una o mas interfaces de extension y se registra con @ExtendWith. Las extensiones pueden mantener estado a traves del ExtensionContext.Store, que proporciona un mapa de almacenamiento con alcance (namespace) controlado.

```java
public class LogExtension implements BeforeEachCallback, AfterEachCallback {

    private static final Logger log = LoggerFactory.getLogger(LogExtension.class);

    @Override
    public void beforeEach(ExtensionContext context) {
        var testClass = context.getRequiredTestClass().getSimpleName();
        var testMethod = context.getRequiredTestMethod().getName();
        log.info(">>> INICIO {}.{}", testClass, testMethod);
    }

    @Override
    public void afterEach(ExtensionContext context) {
        var testClass = context.getRequiredTestClass().getSimpleName();
        var testMethod = context.getRequiredTestMethod().getName();
        var resultado = context.getExecutionException().isPresent() ? "FALLO" : "OK";
        log.info(">>> FIN {}.{} [{}]", testClass, testMethod, resultado);
    }
}
```

Uso de la extension personalizada:

```java
@ExtendWith(LogExtension.class)
class UsuarioServiceTest {
    @Test
    void testAltaUsuario() {
        // LogExtension registrara automaticamente antes y despues
    }
}
```

Las extensiones pueden registrarse tambien de forma global mediante el fichero `META-INF/services/org.junit.jupiter.api.extension.Extension`, lo que evita tener que anotar cada clase de test. Este mecanismo utiliza el ServiceLoader de Java para descubrir y registrar automaticamente las extensiones presentes en el classpath.
