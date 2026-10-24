# Testing - buenas practicas

Probar la capa de persistencia es esencial para garantizar que las consultas devuelven los resultados esperados, que las entidades se mapean correctamente y que las transacciones se comportan segun la especificacion. Sin embargo, los tests de persistencia son intrinsecamente mas lentos y fragiles que los tests unitarios porque dependen de una base de datos. Elegir la estrategia adecuada para cada nivel de test marca la diferencia entre una suite de tests rapida y confiable y una que los desarrolladores terminan ignorando.

## Tests de repositorios con H2

Para tests unitarios de repositorios que verifican consultas JPQL, metodos derivados y mapeo basico, H2 en modo embebido es la opcion mas rapida. Configurado con dialecto compatible con PostgreSQL o MySQL, permite ejecutar tests sin necesidad de infraestructura externa.

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
```

```java
@DataJpaTest
@ActiveProfiles("test")
class PedidoRepositoryTest {

    @Autowired
    private PedidoRepository pedidoRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void deberiaEncontrarPedidoPorEstado() {
        Pedido pedido = new Pedido("PENDIENTE", BigDecimal.valueOf(100));
        entityManager.persistAndFlush(pedido);

        List<Pedido> resultados = pedidoRepository.findByEstado("PENDIENTE");

        assertThat(resultados).hasSize(1);
        assertThat(resultados.get(0).getId()).isEqualTo(pedido.getId());
    }

    @Test
    void deberiaDevolverVacioCuandoNoExisteEstado() {
        List<Pedido> resultados = pedidoRepository.findByEstado("INEXISTENTE");
        assertThat(resultados).isEmpty();
    }

    @Test
    void deberiaContarPedidosActivos() {
        entityManager.persist(new Pedido("ACTIVO", BigDecimal.valueOf(50)));
        entityManager.persist(new Pedido("ACTIVO", BigDecimal.valueOf(75)));
        entityManager.persist(new Pedido("INACTIVO", BigDecimal.valueOf(30)));
        entityManager.flush();

        long count = pedidoRepository.countByEstado("ACTIVO");
        assertThat(count).isEqualTo(2);
    }
}
```

- `@DataJpaTest` configura unicamente los beans de la capa de persistencia (repositorios, EntityManager, DataSource) y aplica `@Transactional` a cada test para que los cambios se reviertan automaticamente al finalizar.
- `TestEntityManager` es un wrapper de `EntityManager` que facilita la preparacion de datos en los tests sin exponer metodos de repositorio.
- El perfil `test` activa H2 en memoria, evitando que los tests impacten en la base de datos de desarrollo.

> `@DataJpaTest` no levanta todo el contexto de Spring, solo los componentes relacionados con JPA. Esto hace que los tests sean mucho mas rapidos que un `@SpringBootTest`.

## Tests de integracion con TestContainers

H2 no es una sustitucion perfecta de la base de datos de produccion. Diferencias en el dialecto SQL, funciones especificas del motor o indices de texto completo pueden provocar que un test pase en H2 y falle en produccion. TestContainers permite levantar contenedores Docker con la base de datos real (PostgreSQL, MySQL, MariaDB) durante la ejecucion de los tests de integracion.

```java
@SpringBootTest
@Testcontainers
class PedidoRepositoryIntegrationTest {

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

    @Autowired
    private PedidoRepository pedidoRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void deberiaPersistirYRecuperarPedido() {
        Pedido pedido = new Pedido("CONFIRMADO", BigDecimal.valueOf(200));
        Pedido guardado = pedidoRepository.save(pedido);

        Optional<Pedido> recuperado = pedidoRepository.findById(guardado.getId());

        assertThat(recuperado).isPresent();
        assertThat(recuperado.get().getEstado()).isEqualTo("CONFIRMADO");
    }
}
```

- `@Testcontainers` activa el soporte de TestContainers en el test.
- `@Container` anota el contenedor que se iniciara antes de los tests y se detendra al finalizar.
- `@DynamicPropertySource` sobrescribe las propiedades de conexion en tiempo de ejecucion para apuntar al contenedor.
- El contenedor se reutiliza entre tests de la misma clase si se configura con `withReuse(true)`.

> TestContainers requiere Docker en el entorno de ejecucion. Para entornos CI sin Docker, se puede configurar un contenedor como servicio en el pipeline o usar una base de datos compartida con `TC_INITSCRIPT` para inicializar el esquema.

## Mock repos en tests de servicio

Cuando se prueba la logica de negocio de un servicio, no interesa probar el comportamiento del repositorio real, sino verificar que el servicio interactua correctamente con sus dependencias. En estos tests se mockean los repositorios y se validan las interacciones.

```java
@ExtendWith(MockitoExtension.class)
class PedidoServiceTest {

    @Mock
    private PedidoRepository pedidoRepository;

    @Mock
    private InventarioRepository inventarioRepository;

    @InjectMocks
    private PedidoService pedidoService;

    @Test
    void deberiaCrearPedidoYReservarStock() {
        CrearPedidoRequest request = new CrearPedidoRequest(1L, List.of(
            new LineaRequest(100L, 2)
        ));
        Pedido pedidoEsperado = new Pedido(1L, "PENDIENTE", BigDecimal.valueOf(200));

        when(pedidoRepository.save(any(Pedido.class))).thenReturn(pedidoEsperado);

        Pedido resultado = pedidoService.crearPedido(request);

        assertThat(resultado.getId()).isEqualTo(1L);
        verify(inventarioRepository).reservarStock(100L, 2);
        verify(pedidoRepository).save(any(Pedido.class));
    }

    @Test
    void deberiaLanzarExcepcionSiPedidoNoExiste() {
        when(pedidoRepository.findById(999L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> pedidoService.cancelarPedido(999L))
            .isInstanceOf(PedidoNoEncontradoException.class)
            .hasMessageContaining("999");
    }
}
```

- `@Mock`: crea un mock del repositorio.
- `@InjectMocks`: inyecta los mocks en el servicio bajo test.
- `verify()`: comprueba que el servicio llamo al repositorio con los parametros esperados.
- No se necesita base de datos ni contexto de Spring. Los tests se ejecutan en milisegundos.

## Datos de prueba consistentes

La preparacion de datos de prueba es una de las partes mas propensas a errores en los tests de persistencia. Usar fabricas de objetos o builders para crear entidades con valores por defecto sensibles mejora la legibilidad y reduce la duplicacion.

```java
public class PedidoTestFactory {

    public static Pedido crearPedidoPendiente() {
        return new Pedido("PENDIENTE", BigDecimal.valueOf(100));
    }

    public static Pedido crearPedidoConCliente(Long clienteId) {
        Pedido pedido = new Pedido("ACTIVO", BigDecimal.valueOf(250));
        pedido.setClienteId(clienteId);
        return pedido;
    }

    public static LineaPedido crearLinea(Long productoId, int cantidad) {
        return new LineaPedido(productoId, cantidad, BigDecimal.valueOf(50));
    }
}

// Uso en test
@Test
void deberiaCalcularTotalDePedido() {
    Pedido pedido = PedidoTestFactory.crearPedidoPendiente();
    pedido.addLinea(PedidoTestFactory.crearLinea(1L, 3));
    pedido.addLinea(PedidoTestFactory.crearLinea(2L, 1));

    entityManager.persistAndFlush(pedido);

    assertThat(pedido.getTotal()).isEqualByComparingTo(BigDecimal.valueOf(200));
}
```

Para escenarios mas complejos se puede usar la libreria `Instancio` o `FixtureFactory`, que generan entidades pobladas con datos aleatorios pero coherentes.

## Tests de transacciones

Verificar el comportamiento transaccional requiere ejecutar operaciones y comprobar que los cambios se revierten cuando ocurre un error. `@DataJpaTest` ya envuelve cada test en una transaccion que hace rollback al final, pero a veces se necesita probar la logica de `@Transactional` explicita.

```java
@SpringBootTest
class TransaccionServiceTest {

    @Autowired
    private PedidoService pedidoService;

    @Autowired
    private PedidoRepository pedidoRepository;

    @Test
    void deberiaHacerRollbackSiFallaElInventario() {
        CrearPedidoRequest request = new CrearPedidoRequest(1L, List.of(
            new LineaRequest(999L, 1) // producto inexistente
        ));

        assertThatThrownBy(() -> pedidoService.crearPedido(request))
            .isInstanceOf(InventarioInsuficienteException.class);

        // Verificar que no quedo ningun pedido persistido
        assertThat(pedidoRepository.count()).isZero();
    }

    @Test
    void deberiaPersistirAuditoriaInclusoConRollback() {
        // El servicio de auditoria usa REQUIRES_NEW
        // por lo que sus cambios persisten aunque el padre haga rollback
        assertThat(eventoRepository.count()).isGreaterThan(0);
    }
}
```

Para probar escenarios de concurrencia (bloqueo optimista, deadlocks) se pueden lanzar multiples hilos que operen sobre la misma entidad simultaneamente:

```java
@Test
void deberiaLanzarOptimisticLockEnActualizacionConcurrente() throws Exception {
    Pedido pedido = entityManager.persistAndFlush(new Pedido("ACTIVO", BigDecimal.valueOf(100)));

    ExecutorService executor = Executors.newFixedThreadPool(2);
    CountDownLatch latch = new CountDownLatch(1);

    Future<?> t1 = executor.submit(() -> {
        latch.await();
        pedidoService.actualizarEstado(pedido.getId(), "CANCELADO");
        return null;
    });

    Future<?> t2 = executor.submit(() -> {
        latch.await();
        pedidoService.actualizarEstado(pedido.getId(), "ENTREGADO");
        return null;
    });

    latch.countDown();
    assertThatThrownBy(() -> { t1.get(); t2.get(); })
        .isInstanceOf(OptimisticLockingFailureException.class);
}
```

Un test de persistencia bien diseñado no solo verifica que el codigo funciona, sino que documenta el comportamiento esperado de las consultas y transacciones, actuando como especificacion ejecutable del modelo de datos.
