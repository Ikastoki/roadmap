# Testing con TestContainers

H2 es adecuado para pruebas rapidas, pero presenta limitaciones significativas: no soporta todas las caracteristicas de los motores de bases de datos de produccion (PostgreSQL, MySQL, Oracle), tiene diferencias en el manejo de tipos de datos, funciones SQL y comportamientos transaccionales. TestContainers resuelve este problema permitiendo ejecutar instancias reales de bases de datos dentro de contenedores Docker durante la ejecucion de los tests, garantizando que las pruebas se ejecuten contra el mismo motor que se utilizara en produccion.

## Dependencias

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

## @Testcontainers y @Container

`@Testcontainers` es una anotacion a nivel de clase que habilita el lifecycle management de los contenedores. `@Container` marca los campos que representan contenedores, indicando a TestContainers como gestionar su ciclo de vida.

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UsuarioRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configurarPropiedades(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.datasource.driver-class-name", postgres::getDriverClassName);
    }

    @Autowired
    private UsuarioRepository usuarioRepository;

    @Test
    void deberiaGuardarYEncontrarUsuario() {
        Usuario usuario = new Usuario("juan@email.com", "Juan", "Perez");
        usuarioRepository.save(usuario);

        Optional<Usuario> encontrado = usuarioRepository.findById(usuario.getId());

        assertThat(encontrado).isPresent();
        assertThat(encontrado.get().getEmail()).isEqualTo("juan@email.com");
    }
}
```

El contenedor se arranca antes de ejecutar cualquier test de la clase y se detiene al finalizar todos. `@DynamicPropertySource` sobreescribe las propiedades de conexion para que Spring Boot use la URL, usuario y contraseña del contenedor.

> Los campos anotados con `@Container` deben ser `static` si se quiere que el contenedor sea compartido entre todos los tests de la clase. Si no son static, cada test crea su propio contenedor, lo que incrementa significativamente el tiempo de ejecucion.

## PostgreSQLContainer

TestContainers proporciona contenedores especializados para los principales motores de base de datos:

```java
// PostgreSQL
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

// MySQL
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

// MariaDB
static MariaDBContainer<?> mariadb = new MariaDBContainer<>("mariadb:10.11");

// MSSQL
static MSSQLServerContainer<?> sqlserver = new MSSQLServerContainer<>("mcr.microsoft.com/mssql/server:2022-latest");
```

Cada contenedor expone automaticamente el puerto, la URL JDBC, el usuario y la contraseña por defecto, eliminando la necesidad de configuracion manual.

## MongoDB con TestContainers

Para testing de persistencia con MongoDB:

```java
@DataMongoTest
@Testcontainers
class UsuarioMongoRepositoryTest {

    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:6.0");

    @DynamicPropertySource
    static void configurarPropiedades(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongodb::getReplicaSetUrl);
    }

    @Autowired
    private UsuarioMongoRepository usuarioRepository;
}
```

## Reutilizacion de contenedores

TestContainers puede reutilizar contenedores entre ejecuciones de tests para reducir el tiempo de arranque. Se configura mediante propiedad o variable de entorno:

```properties
testcontainers.reuse.enable=true
```

O en `~/.testcontainers.properties`:

```properties
reuse.enable=true
```

La reutilizacion requiere anadir `.withReuse(true)` al contenedor:

```java
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
    .withReuse(true);
```

> La reutilizacion es util en desarrollo local pero debe desactivarse en entornos CI para garantizar que cada ejecucion parte de un estado limpio.

## Ventajas frente a H2

Las pruebas con TestContainers son 100% compatibles con el entorno de produccion. No hay diferencias de dialecto SQL, comportamiento de tipos de datos, funciones especificas del motor ni limitaciones del modo embebido. Esto elimina la categoria de errores que surgen cuando "en H2 funciona pero en PostgreSQL no".

El coste es el tiempo de arranque del contenedor (tipicamente 5-15 segundos la primera vez) y el mayor consumo de recursos. Una estrategia comun es usar H2 para pruebas unitarias rapidas de repositorios simples y TestContainers para pruebas de integracion que verifican comportamientos especificos del motor de base de datos.

## Imagenes personalizadas

Para bases de datos con configuracion especifica, se puede utilizar `GenericContainer`:

```java
@Container
static GenericContainer<?> postgresCustom = new GenericContainer<>(
    DockerImageName.parse("postgis/postgis:15-3.4")
)
    .withEnv("POSTGRES_DB", "testdb")
    .withEnv("POSTGRES_USER", "test")
    .withEnv("POSTGRES_PASSWORD", "test")
    .withExposedPorts(5432);
```
