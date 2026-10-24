# Testing Strategy

Una estrategia de testing solida en Spring Boot se organiza en capas: pruebas unitarias rapidas que validan la logica de negocio aislada, pruebas de integracion que verifican la interaccion entre componentes, y pruebas end-to-end que validan el comportamiento completo de la aplicacion. Cada tipo de prueba tiene un scopo diferente, un nivel de ejecucion diferente y un trade-off entre velocidad y cobertura.

## Unit tests con Mockito

Las pruebas unitarias validan la logica de negocio de un componente de forma aislada, simulando todas las dependencias externas con mocks. Son rapidas (milisegundos) y se ejecutan sin levantar el contexto de Spring.

```java
@ExtendWith(MockitoExtension.class)
class PedidoServiceTest {

    @Mock
    private PedidoRepository pedidoRepository;

    @Mock
    private ProductoService productoService;

    @Mock
    private NotificacionService notificacionService;

    @InjectMocks
    private PedidoServiceImpl pedidoService;

    @Test
    @DisplayName("Crear pedido con items validos")
    void crearPedido_conItemsValidos_retornaPedidoCreado() {
        // Arrange
        CrearPedidoDTO dto = new CrearPedidoDTO(
            List.of(new ItemPedidoDTO(1L, 2), new ItemPedidoDTO(2L, 1))
        );

        when(productoService.obtenerPorId(1L))
            .thenReturn(new ProductoDTO(1L, "Producto A", new BigDecimal("25.00")));
        when(productoService.obtenerPorId(2L))
            .thenReturn(new ProductoDTO(2L, "Producto B", new BigDecimal("50.00")));
        when(pedidoRepository.save(any(Pedido.class)))
            .thenAnswer(invocation -> {
                Pedido p = invocation.getArgument(0);
                p.setId(1L);
                return p;
            });

        // Act
        PedidoDTO resultado = pedidoService.crear(dto);

        // Assert
        assertNotNull(resultado);
        assertEquals(1L, resultado.getId());
        assertEquals(new BigDecimal("100.00"), resultado.getTotal());
        verify(pedidoRepository).save(any(Pedido.class));
        verify(notificacionService).enviarConfirmacion(any());
    }

    @Test
    @DisplayName("Crear pedido sin items lanza excepcion")
    void crearPedido_sinItems_lanzaExcepcion() {
        CrearPedidoDTO dto = new CrearPedidoDTO(List.of());

        assertThrows(BusinessException.class,
            () -> pedidoService.crear(dto));
    }
}
```

## Integration tests con @SpringBootTest

`@SpringBootTest` levanta el contexto completo de Spring, incluyendo beans, configuraciones y dependencias inyectadas. Son mas lentas pero verifican que los componentes se integran correctamente.

```java
@SpringBootTest
@ActiveProfiles("test")
@Transactional
class PedidoServiceIntegrationTest {

    @Autowired
    private PedidoService pedidoService;

    @Autowired
    private PedidoRepository pedidoRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    @DisplayName("Guardar y recuperar pedido con transaccion")
    void guardarYRecuperarPedido() {
        // Arrange
        Producto producto = new Producto(null, "Laptop", new BigDecimal("999.99"));
        entityManager.persist(producto);

        CrearPedidoDTO dto = new CrearPedidoDTO(
            List.of(new ItemPedidoDTO(producto.getId(), 1))
        );

        // Act
        PedidoDTO creado = pedidoService.crear(dto);
        PedidoDTO recuperado = pedidoService.obtenerPorId(creado.getId());

        // Assert
        assertNotNull(recuperado);
        assertEquals(creado.getId(), recuperado.getId());
        assertEquals(1, recuperado.getItems().size());
    }
}
```

> `@Transactional` en pruebas de integracion ejecuta cada test dentro de una transaccion que se revierte automaticamente al final, garantizando que los datos de prueba no contaminen otros tests.

## Web tests con @WebMvcTest

`@WebMvcTest` levanta solo la capa web (controllers, filters, exception handlers) con mocks para las dependencias de servicio. Es ideal para probar endpoints HTTP sin levantar la capa de persistencia.

```java
@WebMvcTest(UsuarioController.class)
class UsuarioControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UsuarioService usuarioService;

    @Test
    @DisplayName("GET /api/usuarios/{id} retorna 200 con usuario")
    void obtenerUsuario_existente_retorna200() throws Exception {
        UsuarioDTO usuario = new UsuarioDTO(1L, "Juan", "juan@ejemplo.com");
        when(usuarioService.obtenerPorId(1L)).thenReturn(usuario);

        mockMvc.perform(get("/api/v1/usuarios/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.nombre").value("Juan"))
            .andExpect(jsonPath("$.email").value("juan@ejemplo.com"));
    }

    @Test
    @DisplayName("GET /api/usuarios/{id} retorna 404 cuando no existe")
    void obtenerUsuario_inexistente_retorna404() throws Exception {
        when(usuarioService.obtenerPorId(999L))
            .thenThrow(new ResourceNotFoundException("Usuario", 999L));

        mockMvc.perform(get("/api/v1/usuarios/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("Recurso no encontrado"));
    }
}
```

## Data tests con @DataJpaTest

`@DataJpaTest` configura un entorno de prueba optimizado para repositorios JPA con una base de datos embebida (H2 o HSQL) y `TestEntityManager` para operaciones directas sobre el persistence context.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UsuarioRepositoryTest {

    @Autowired
    private UsuarioRepository usuarioRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void buscarPorEmail_existente_retornaUsuario() {
        Usuario usuario = new Usuario(null, "Ana", "ana@test.com", true);
        entityManager.persistAndFlush(usuario);

        Optional<Usuario> resultado = usuarioRepository.findByEmail("ana@test.com");

        assertTrue(resultado.isPresent());
        assertEquals("Ana", resultado.get().getNombre());
    }

    @Test
    void buscarPorEmail_inexistente_retornaVacio() {
        Optional<Usuario> resultado = usuarioRepository.findByEmail("noexiste@test.com");
        assertTrue(resultado.isEmpty());
    }
}
```

## TestContainers

TestContainers proporciona contenedores Docker ligeros para pruebas de integracion con bases de datos reales, colas de mensajes y otros servicios. Elimina la diferencia entre el entorno de prueba y produccion.

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
class PedidoRepositoryContainerTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
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

    @Test
    void guardarPedido_enPostgresReal() {
        Pedido pedido = new Pedido(null, new BigDecimal("100.00"),
            PedidoEstado.PENDIENTE, LocalDateTime.now());
        Pedido guardado = pedidoRepository.save(pedido);

        assertNotNull(guardado.getId());
        assertEquals(PedidoEstado.PENDIENTE, guardado.getEstado());
    }
}
```

> TestContainers ejecuta contenedores Docker reales, por lo que Docker debe estar disponible en la maquina de ejecucion. En pipelines CI/CD, se recomienda utilizar Docker-in-Docker o Docker socket mounting.

