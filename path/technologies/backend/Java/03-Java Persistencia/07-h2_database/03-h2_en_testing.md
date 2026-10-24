# H2 en testing

El uso de bases de datos embebidas para testing es una practica establecida en el ecosistema Java. H2 es la opcion mas extendida por su velocidad, facilidad de configuracion y nula dependencia de infraestructura externa. Sin embargo, conviene conocer tanto sus ventajas como sus limitaciones para decidir cuando es adecuado utilizarlo y cuando conviene recurrir a alternativas como TestContainers.

## H2 vs TestContainers

TestContainers permite levantar instancias reales de bases de datos (PostgreSQL, MySQL, MariaDB) dentro de contenedores Docker durante la ejecucion de los tests. La principal diferencia con H2 radica en que TestContainers ejecuta el motor real de base de datos, mientras que H2 emula el comportamiento mediante su propio motor.

| Aspecto | H2 | TestContainers |
|---------|----|-----------------|
| Velocidad de arranque | Milisegundos | Segundos (descarga de imagen + inicio contenedor) |
| Consumo de recursos | Minimo | Moderado (Docker + contenedor) |
| Fidelidad dialecto SQL | Parcial | Total |
| Requisitos | Solo JVM | Docker |
| Comportamiento transaccional | Aproximado | Identico al real |
| Funciones nativas de BD | Emuladas | Reales |

## Ventajas de H2 en testing

El principal beneficio de H2 es la velocidad. Los tests que usan H2 arrancan en milisegundos y no requieren Docker, lo que los hace ideales para el ciclo de desarrollo local y para la mayoria de pruebas unitarias.

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class RepositorioTest {

    @Autowired
    private Repositorio repositorio;

    @Test
    void deberiaGuardarEntidad() {
        Entidad e = new Entidad("test");
        repositorio.save(e);
        assertThat(repositorio.findById(e.getId())).isPresent();
    }
}
```

La anotacion `@AutoConfigureTestDatabase` con `Replace.ANY` reemplaza automaticamente cualquier DataSource configurado por uno de H2 en memoria. Es la forma mas rapida de configurar testing con H2 en Spring Boot.

Para proyectos que no usan Spring Boot, la configuracion manual es igualmente sencilla:

```java
@BeforeEach
void setUp() throws SQLException {
    DataSource ds = new EmbeddedDatabaseBuilder()
        .setType(EmbeddedDatabaseType.H2)
        .addScript("classpath:schema.sql")
        .addScript("classpath:test-data.sql")
        .build();
}
```

## Limitaciones de H2

H2 no es un reemplazo perfecto de PostgreSQL ni de MySQL. Existen diferencias sutiles que pueden provocar que los tests pasen con H2 pero fallen en produccion.

Casos frecuentes de divergencia:
- H2 no soporta `JSONB` de PostgreSQL ni los operadores de JSON completos.
- Las funciones de agregacion y ventana tienen diferencias en el manejo de nulos.
- Los tipos `UUID` se almacenan de forma distinta (binario en H2 vs texto en PostgreSQL).
- Las secuencias y generadores de identidad se comportan de manera diferente bajo concurrencia.
- Las restricciones `CHECK` y `UNIQUE` pueden validarse en momentos distintos.

> Si la aplicacion utiliza caracteristicas especificas de PostgreSQL (tipos como `JSONB`, `ARRAY`, `TSVECTOR`, indices GIN, funciones de extension), H2 no podra emularlas correctamente. En esos casos, TestContainers es la unica opcion viable para tests fiables.

## Uso de @Sql para carga de datos

Spring proporciona la anotacion `@Sql` para ejecutar scripts SQL antes o despues de los tests. Es especialmente util con H2 para inicializar el esquema y los datos de prueba.

```java
@SpringBootTest
@Sql(scripts = "/sql/insertar-clientes.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "/sql/limpiar.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class ClienteRepositoryTest {

    @Test
    @Sql("/sql/insertar-cliente-especifico.sql")
    void deberiaBuscarClientePorEmail() {
        // ...
    }
}
```

Los scripts se ejecutan en el orden declarado. Es recomendable que cada test tenga un script de limpieza posterior (`AFTER_TEST_METHOD`) para evitar acoplamiento entre pruebas.

```sql
-- /sql/limpiar.sql
SET REFERENTIAL_INTEGRITY FALSE;
TRUNCATE TABLE clientes RESTART IDENTITY;
TRUNCATE TABLE pedidos RESTART IDENTITY;
SET REFERENTIAL_INTEGRITY TRUE;
```

La combinacion de `TRUNCATE` con `RESTART IDENTITY` y la desactivacion temporal de restricciones referenciales garantiza un estado limpio entre tests.

## Estrategias de testing recomendadas

Para la mayoria de proyectos, la combinacion optima es:
- Tests unitarios de repositorio con H2 en memoria para validar consultas basicas y operaciones CRUD.
- Tests de integracion de servicios con TestContainers para validar comportamientos que dependen del dialecto real de la base de datos.
- Tests de contratos de API con H2 por su rapidez de ejecucion.

```java

@SpringBootTest
@Testcontainers
class PedidoServicioIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

Esta estrategia dual aprovecha la velocidad de H2 para la mayoria de tests y reserva TestContainers para aquellos escenarios donde la fidelidad dialectica es critica.
