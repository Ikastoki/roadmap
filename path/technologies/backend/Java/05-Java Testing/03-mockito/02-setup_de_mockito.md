# Setup de Mockito

La integracion de Mockito en un proyecto Java comienza con la inclusion de las dependencias correctas. Para proyectos Maven, el artefacto principal es `mockito-core` o `mockito-inline` si se requiere soporte para metodos estaticos y final. La configracion tipica en `pom.xml` es:

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.12.0</version>
    <scope>test</scope>
</dependency>
```

Para proyectos Gradle, la dependencia equivalente:

```groovy
testImplementation 'org.mockito:mockito-core:5.12.0'
```

Mockito se integra con JUnit 5 mediante la extension `MockitoExtension`, que gestiona el ciclo de vida de los mocks automaticamente. La anotacion `@ExtendWith` registra la extension en la clase de prueba:

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class ServicioUsuarioTest {

}
```

Esta extension elimina la necesidad de invocar manualmente `MockitoAnnotations.openMocks()`, aunque esta opcion sigue disponible para escenarios donde no se usa JUnit 5 o se requiere control explicito. `MockitoAnnotations.openMocks()` procesa las anotaciones de Mockito y debe llamarse antes de cada prueba, tipicamente en un metodo `@BeforeEach`:

```java
class ServicioUsuarioTest {

    @Mock
    private RepositorioUsuario repositorio;

    @InjectMocks
    private ServicioUsuario servicio;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }
}
```

Las anotaciones principales que Mockito pone a disposicion son cuatro:

`@Mock` crea una instancia simulada de una clase o interfaz. Todos sus metodos devuelven valores por defecto (cero, false, colecciones vacias, null) hasta que se configuren con stubbing. `@Spy` crea un espia que envuelve una instancia real, preservando el comportamiento original salvo para los metodos que se stubbeen explicitamente. `@InjectMocks` intenta inyectar los mocks y spies anotados en la instancia de la clase bajo prueba, explorando el constructor, setters y campos privados por ese orden. `@Captor` declara un captor de argumentos de tipo `ArgumentCaptor<T>` que se inicializa automaticamente.

> La inyeccion con `@InjectMocks` no siempre es fiable cuando hay ambiguedad entre multiples mocks del mismo tipo. En esos casos es preferible la inyeccion manual en el constructor.

El orden de prioridad que sigue `MockitoExtension` para inicializar las anotaciones es equivalente a `MockitoAnnotations.openMocks()`, garantizando que todos los campos anotados esten listos antes de ejecutar cualquier metodo `@Test`. La extension tambien maneja la validacion de uso: si se invoca `when()` sin configurar un mock, o se usa `verify()` incorrectamente, la extension reporta errores con mensajes descriptivos que senalan la linea exacta del problema.

```java
@ExtendWith(MockitoExtension.class)
class ServicioFacturaTest {

    @Mock
    private CalculadoraImpuestos calculadora;

    @Spy
    private AlmacenFacturas almacen = new AlmacenFacturas();

    @InjectMocks
    private ServicioFactura servicio;

    @Captor
    private ArgumentCaptor<Factura> facturaCaptor;

    @Test
    void testEmision() {
        // ...
    }
}
```

En proyectos que usan TestNG en lugar de JUnit 5, la inicializacion se realiza con `MockitoTestNGListener` o mediante `MockitoAnnotations.openMocks(this)` en un metodo `@BeforeMethod`. La flexibilidad de Mockito permite adaptarse a cualquier framework de testing sin cambios en la logica de las pruebas.
