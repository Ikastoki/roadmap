# Four-Phase Test

El patron Four-Phase Test es una generalizacion de AAA que anade una cuarta fase explicita: Teardown. Mientras AAA se centra en la estructura logica de la verificacion, Four-Phase describe el ciclo de vida completo de un test, incluyendo la limpieza posterior a la ejecucion.

## Setup

La fase Setup corresponde a Arrange en AAA. Se crean los objetos, se configuran los datos y se prepara el entorno para la ejecucion del test. Esta fase puede ejecutarse una vez por test o una vez por clase, dependiendo de si los objetos pueden ser reutilizados.

```java
class ShoppingCartTest {

    private ShoppingCart cart;
    private Product product;

    @BeforeEach
    void setUp() {
        // Setup: preparar el escenario
        cart = new ShoppingCart();
        product = new Product("SKU-001", "Laptop", 1200.0);
    }
```

La fase Setup puede delegar en metodos factoria o builders para simplificar la creacion de objetos complejos. El objetivo es que el cuerpo del test sea legible sin distraerse con la inicializacion.

## Exercise

Exercise es la fase Act de AAA. Se ejecuta la operacion bajo prueba. Esta fase debe ser lo mas breve posible, idealmente una unica llamada.

```java
    @Test
    void shouldAddProductToCart() {
        // Exercise
        cart.addProduct(product, 2);
```

Si la operacion bajo prueba requiere multiples pasos, el test podria estar violando el principio de responsabilidad unica o podria necesitar una abstraccion mas rica.

## Verify

Verify es la fase Assert. Se comprueba que el resultado de la operacion coincide con lo esperado. Las verificaciones deben ser especificas del comportamiento que se esta probando y no incluir aserciones sobre aspectos no relacionados.

```java
        // Verify
        assertEquals(2, cart.getItemCount());
        assertEquals(2400.0, cart.getTotal(), 0.001);
    }
```

## Teardown

La fase Teardown restaura el estado inicial del sistema despues de la ejecucion del test. En la mayoria de los tests unitarios de Java, el Teardown es irrelevante porque los objetos se crean y destruyen dentro del mismo test y el garbage collector se encarga de la limpieza. Sin embargo, en tests que interactuan con recursos externos (ficheros temporales, conexiones de base de datos, servicios en red), el Teardown es critico.

```java
class FileProcessorTest {

    private Path tempFile;

    @BeforeEach
    void setUp() throws IOException {
        tempFile = Files.createTempFile("test-", ".txt");
    }

    @Test
    void shouldWriteContentToFile() throws IOException {
        FileProcessor processor = new FileProcessor();
        processor.write(tempFile, "Hello, World!");

        assertEquals("Hello, World!", Files.readString(tempFile));
    }

    @AfterEach
    void tearDown() throws IOException {
        // Teardown: eliminar el fichero temporal
        Files.deleteIfExists(tempFile);
    }
}
```

> En tests que abren conexiones de red, hilos o ficheros, el Teardown no es opcional: es la garantia de que un test no contamina al siguiente.

## Lifecycle completo de un test

El ciclo de vida completo ejecuta siempre Setup y Teardown, y condicionalmente Exercise y Verify (si el test falla en Exercise, nunca llega a Verify, pero Teardown se ejecuta igual):

1. `@BeforeAll` (una vez por clase)
2. `@BeforeEach` (antes de cada test)
3. Exercise + Verify (el test en si)
4. `@AfterEach` (despues de cada test)
5. `@AfterAll` (una vez por clase)

```java
class DatabaseTest {

    private Connection connection;

    @BeforeAll
    static void initDatabase() {
        // Setup global: crear esquema de base de datos
    }

    @BeforeEach
    void setUp() {
        connection = Database.connect("jdbc:h2:mem:test");
        connection.beginTransaction();
    }

    @Test
    void shouldPersistUser() {
        UserRepository repo = new UserRepository(connection);
        repo.save(new User("john@example.com", "John"));

        User found = repo.findByEmail("john@example.com");
        assertEquals("John", found.getName());
    }

    @AfterEach
    void tearDown() {
        connection.rollback(); // nunca persiste cambios
        connection.close();
    }

    @AfterAll
    static void cleanupDatabase() {
        // Teardown global: eliminar esquema
    }
}
```

El patron Four-Phase es particularmente util en tests de integracion, donde el estado global (base de datos, sistema de ficheros, colas de mensajes) debe ser gestionado explicitamente para garantizar la independencia entre tests.
