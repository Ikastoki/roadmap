# Datos de prueba

La gestion de datos de prueba es un aspecto crucial en el testing de persistencia. Unos datos de prueba bien disenados permiten verificar casos limite, validar consultas y asegurar que el comportamiento de la aplicacion es correcto bajo diferentes condiciones. Spring Boot proporciona varias herramientas para definir y cargar datos de prueba, desde anotaciones declarativas hasta la generacion programatica con librerias especializadas.

## @Sql

La anotacion `@Sql` permite ejecutar scripts SQL antes o despues de un metodo de test. Los scripts se ejecutan en el contexto transaccional del test, por lo que los cambios se revierten automaticamente al finalizar.

```java
@DataJpaTest
class UsuarioRepositoryTest {

    @Autowired
    private UsuarioRepository usuarioRepository;

    @Test
    @Sql("/datos/prueba/usuarios_iniciales.sql")
    void deberiaEncontrarUsuariosIniciales() {
        List<Usuario> usuarios = usuarioRepository.findAll();
        assertThat(usuarios).hasSize(2);
    }
}
```

El script `usuarios_iniciales.sql` se ubica en `src/test/resources/datos/prueba/`:

```sql
INSERT INTO usuarios (email, nombre, apellido) VALUES ('admin@email.com', 'Admin', 'Root');
INSERT INTO usuarios (email, nombre, apellido) VALUES ('user@email.com', 'Usuario', 'Normal');
```

`@Sql` acepta parametros adicionales para controlar la ejecucion:

```java
@Test
@Sql(
    scripts = "/datos/prueba/usuarios_iniciales.sql",
    config = @SqlConfig(
        encoding = "UTF-8",
        separator = ";",
        commentPrefix = "--",
        transactionMode = SqlConfig.TransactionMode.ISOLATED
    ),
    executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD
)
void testConConfiguracionPersonalizada() {
    // ...
}
```

`executionPhase` determina si el script se ejecuta antes (`BEFORE_TEST_METHOD`) o despues (`AFTER_TEST_METHOD`) del test. El modo `AFTER_TEST_METHOD` es util para limpiar datos creados durante el test.

## @SqlGroup

`@SqlGroup` agrupa multiples anotaciones `@Sql` cuando se necesita ejecutar varios scripts en fases diferentes:

```java
@Test
@SqlGroup({
    @Sql(scripts = "/datos/prueba/limpiar.sql",
         executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD),
    @Sql(scripts = {"/datos/prueba/usuarios.sql", "/datos/prueba/pedidos.sql"},
         executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD),
    @Sql(scripts = "/datos/prueba/limpiar.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
})
void testConMultiplesScripts() {
    // ...
}
```

`@SqlGroup` es util cuando se necesita una secuencia compleja de inicializacion y limpieza, o cuando diferentes conjuntos de datos provienen de distintos archivos.

> Los scripts SQL se ejecutan en el orden en que aparecen dentro de la anotacion `@Sql`. Si hay dependencias entre scripts (por ejemplo, claves foraneas), el orden debe reflejar la jerarquia de las tablas.

## Test fixtures con clases Java

Para escenarios complejos donde los scripts SQL resultan insuficientes (datos con relaciones, fechas calculadas, valores condicionales), se pueden construir datos de prueba programaticamente mediante un builder o un fixture factory:

```java
public class UsuarioFixture {

    public static Usuario crearUsuarioPorDefecto() {
        return Usuario.builder()
            .email("default@email.com")
            .nombre("Default")
            .apellido("User")
            .activo(true)
            .fechaCreacion(LocalDateTime.now())
            .build();
    }

    public static Usuario crearUsuarioInactivo() {
        return Usuario.builder()
            .email("inactivo@email.com")
            .nombre("Inactivo")
            .apellido("User")
            .activo(false)
            .fechaCreacion(LocalDateTime.now().minusDays(30))
            .build();
    }

    public static Pedido crearPedidoConUsuario(Usuario usuario) {
        return Pedido.builder()
            .usuario(usuario)
            .descripcion("Pedido de prueba")
            .total(new BigDecimal("100.00"))
            .fecha(LocalDateTime.now())
            .build();
    }
}
```

Y utilizarlo en el test:

```java
@Test
void deberiaCrearPedidoParaUsuario() {
    Usuario usuario = usuarioRepository.save(UsuarioFixture.crearUsuarioPorDefecto());
    Pedido pedido = PedidoFixture.crearPedidoConUsuario(usuario);
    pedido = pedidoRepository.save(pedido);

    assertThat(pedido.getId()).isNotNull();
    assertThat(pedido.getUsuario().getEmail()).isEqualTo("default@email.com");
}
```

## Faker para datos aleatorios

La libreria Faker (Java Faker o Instancio) genera datos realistas y variados para poblar la base de datos de prueba:

```java
<dependency>
    <groupId>net.datafaker</groupId>
    <artifactId>datafaker</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import net.datafaker.Faker;

class UsuarioFixture {

    private static final Faker faker = new Faker(new Locale("es"));

    public static Usuario crearUsuarioAleatorio() {
        return Usuario.builder()
            .email(faker.internet().emailAddress())
            .nombre(faker.name().firstName())
            .apellido(faker.name().lastName())
            .telefono(faker.phoneNumber().phoneNumber())
            .activo(faker.bool().bool())
            .fechaCreacion(faker.date().past(365, java.util.concurrent.TimeUnit.DAYS)
                .toLocalDateTime().atStartOfDay(ZoneId.systemDefault()).toLocalDateTime())
            .build();
    }
}
```

Faker genera datos que se asemejan a datos reales, lo que permite detectar problemas que no aparecerian con datos triviales como "test1", "test2". Por ejemplo, diferencias en la longitud de cadenas, caracteres especiales en nombres, o formatos de email variados.

## Instancio para poblacion masiva

Instancio es una libreria que pobla automaticamente objetos Java con datos aleatorios, incluyendo colecciones y relaciones:

```java
<dependency>
    <groupId>org.instancio</groupId>
    <artifactId>instancio-junit5</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@Test
void deberiaGuardarMultiplesUsuarios() {
    List<Usuario> usuarios = Instancio.ofList(Usuario.class)
        .size(100)
        .set(field(Usuario::getActivo), true)
        .create();

    usuarioRepository.saveAll(usuarios);

    List<Usuario> encontrados = usuarioRepository.findByActivoTrue();
    assertThat(encontrados).hasSize(100);
}
```

Instancio permite configurar reglas de generacion, ignorar ciertos campos y establecer valores fijos para propiedades especificas.

## Estrategias de datos de prueba

La eleccion entre scripts SQL, fixtures Java o generacion aleatoria depende del tipo de test. Para tests que verifican consultas especificas con valores conocidos, los scripts SQL son ideales porque el resultado esperado es deterministico. Para tests de integracion que ejercitan la logica de negocio con multiples variaciones, los fixtures programaticos ofrecen flexibilidad. Para pruebas de rendimiento o volumen, la generacion aleatoria con Faker o Instancio permite crear conjuntos de datos grandes con poco codigo.
