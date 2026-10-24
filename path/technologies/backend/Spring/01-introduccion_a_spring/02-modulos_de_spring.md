# Modulos de Spring

Spring Framework esta compuesto por un conjunto de modulos que, aunque pueden utilizarse de forma independiente, estan disenos para trabajar de forma coordinada. Cada modulo aborda un problema especifico dentro del ciclo de vida de una aplicacion Java, desde la creacion y configuracion de objetos hasta la exposicion de endpoints REST, pasando por el acceso a datos y la gestion de transacciones. Comprender estos modulos y sus relaciones es esencial para diseñar soluciones correctamente, ya que la eleccion de uno u otro modulo determina la arquitectura de la capa correspondiente.

## spring-core y spring-beans: el nucleo del contenedor

Los modulos `spring-core` y `spring-beans` forman la base sobre la que se construye todo el ecosistema. `spring-core` proporciona las utilidades fundamentales del framework: clases de carga de recursos, internacionalizacion (i18n), eventos y el mecanismo de conversión de tipos (Type Conversion). `spring-beans` es responsable de la definicion, instanciacion, configuracion y ciclo de vida de los objetos gestionados por el contenedor, comunmente denominados "beans".

El contenedor IoC (Inversion of Control) implementado por estos modulos es el corazon de Spring. Este contenedor lee metadatos de configuracion (ya sean anotaciones Java, XML o codigo de configuracion) y los utiliza para crear, ensamblar y gestionar el ciclo de vida completo de los beans. La interfaz principal es `BeanFactory`, que ofrece funcionalidad basica de contenedor con lazy loading. La extension mas completa es `ApplicationContext`, que anade capacidades como resolución de mensajes de internacionalizacion, publicacion de eventos y carga temprana de beans (eager loading).

```java
// BeanFactory - contenedor basico, lazy loading
BeanFactory factory = new DefaultListableBeanFactory();
// Los beans se instancian bajo demanda

// ApplicationContext - contenedor completo, eager loading
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// Todos los beans singleton se instancian al arrancar
UserService service = context.getBean(UserService.class);
```

> La distincion entre `BeanFactory` y `ApplicationContext` es historica. En la practica moderna, `ApplicationContext` es la interfaz que se utiliza en el 99% de los casos. `BeanFactory` solo tiene sentido en escenarios donde el consumo de memoria es critico y se necesita lazy loading estricto.

## spring-context: el marco de referencia

`spring-context` construye sobre `spring-beans` para proporcionar un entorno de referencia (ApplicationContext) mas rico. Este modulo es el que introduce las anotaciones de configuracion como `@Configuration`, `@Component`, `@Bean`, `@ComponentScan` y `@PropertySource`. Tambien incluye soporte para internacionalizacion a traves de `MessageSource`, publicacion y subscripcion de eventos via `ApplicationEventPublisher`, y carga de recursos externos.

El `ApplicationContext` proporcionado por `spring-context` es el que se utiliza en practicamente todas las aplicaciones Spring. Representa una jerarquia de contenedores que puede incluir un contenedor padre y multiples contenedores hijos, permitiendo un aislamiento parcial de beans entre distintas capas de la aplicacion.

```java
@Configuration
@ComponentScan("com.example")
@PropertySource("classpath:application.properties")
public class AppConfig {
    // La anotacion @Configuration indica que esta clase
    // es fuente de definiciones de beans
}
```

## spring-aop: programacion orientada a aspectos

`spring-aop` implementa un subconjunto funcional de AOP (Aspect-Oriented Programming) compatible con AspectJ. Este modulo permite separar concerns transversales —como logging, seguridad, caching y transacciones— del codigo de negocio principal. Spring AOP opera mediante proxies: el contenedor genera dinamicamente una subclase (proxy CGLIB) o una implementacion de interfaz (JDK dynamic proxy) que intercepta las llamadas a los metodos del bean y anade comportamiento adicional antes, despues o alrededor de la ejecucion del metodo original.

```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Ejecutando: " + joinPoint.getSignature().getName());
    }

    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void logAfter(JoinPoint joinPoint, Object result) {
        System.out.println("Completado: " + joinPoint.getSignature().getName() + " -> " + result);
    }
}
```

La integracion de `spring-aop` con otros modulos es profunda. `spring-tx` (transacciones), `spring-security` (autenticacion/autorizacion) y `spring-cache` (caching) utilizan internamente AOP para implementar su comportamiento declarativo. Cuando anades `@Transactional` a un metodo, Spring AOP envuelve la llamada en un proxy que inicia, confirma o revierte la transaccion de forma transparente.

## spring-expression (SpEL): lenguaje de expresiones

`spring-expression` proporciona Spring Expression Language (SpEL), un lenguaje de expresiones potente que permite evaluar dinamicamente valores en tiempo de ejecucion. SpEL se utiliza extensamente en la configuracion de beans, en la evaluacion de atributos de anotaciones y en plantillas de vistas como Thymeleaf.

```java
// SpEL en anotaciones
@Value("#{systemProperties['user.home']}")
private String userHome;

@Value("#{beanService.calculateScore() * 1.1}")
private double adjustedScore;

// SpEL programatico
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'.concat('!')");
String message = (String) exp.getValue(); // "Hello World!"
```

SpEL soporta operaciones aritmeticas, logicas, de comparacion, acceso a propiedades de objetos, invocacion de metodos, operaciones sobre colecciones y expresiones condicionales. Es significativamente mas potente que las expresiones usadas en OGNL o JSP EL.

## spring-jdbc, spring-orm y spring-tx: acceso a datos y transacciones

Estos tres modulos trabajan en conjunto para gestionar la capa de persistencia. `spring-jdbc` simplifica el uso de JDBC puro al eliminar el boilerplate de apertura/cierre de conexiones, manejo de excepciones y mapeo de resultados. La clase `JdbcTemplate` encapsula las operaciones repetitivas y traduce las checked exceptions de JDBC a unchecked exceptions del hierarchy `DataAccessException` de Spring.

```java
@Repository
public class UsuarioJdbcRepository {

    private final JdbcTemplate jdbcTemplate;

    public UsuarioJdbcRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Optional<Usuario> findById(Long id) {
        String sql = "SELECT id, nombre, email FROM usuarios WHERE id = ?";
        List<Usuario> resultados = jdbcTemplate.query(sql, new UsuarioRowMapper(), id);
        return resultados.stream().findFirst();
    }

    public void save(Usuario usuario) {
        jdbcTemplate.update(
            "INSERT INTO usuarios (nombre, email) VALUES (?, ?)",
            usuario.getNombre(), usuario.getEmail()
        );
    }
}
```

`spring-orm` proporciona integracion con Object-Relational Mapping frameworks como Hibernate, EclipseLink y JPA. La clase `LocalContainerEntityManagerFactoryBean` es la forma estandar de configurar un `EntityManagerFactory` en Spring, mientras que `HibernateTemplate` y `JpaTemplate` simplifican las operaciones CRUD.

`spring-tx` implementa el modelo de gestion de transacciones de Spring. La anotacion `@Transactional` permite declarar transacciones de forma programatica o declarativa, con soporte para propagation (REQUIRED, REQUIRES_NEW, NESTED, etc.), isolation levels, rollback rules y timeouts. Detras de esta anotacion, Spring AOP genera un proxy que gestiona el ciclo de vida de la transaccion.

```java
@Service
@Transactional
public class PedidoService {

    private final PedidoRepository pedidoRepository;
    private final InventarioService inventarioService;

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public Pedido crearPedido(PedidoRequest request) {
        // Esta operacion se ejecuta en una transaccion nueva
        inventarioService.reservarStock(request.getItems());
        Pedido pedido = new Pedido(request);
        return pedidoRepository.save(pedido);
    }
}
```

## spring-web y spring-webmvc: capa web

`spring-web` proporciona las utilidades basicas para web: soporte para multipart file upload, inicializacion del `DispatcherServlet` y abstracciones para servlets y filters. `spring-webmvc` implementa el patron Model-View-Controller sobre Servlet API, con el `DispatcherServlet` como punto de entrada unico que delega las peticiones a controllers, handlers y view resolvers.

```java
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioController {

    private final UsuarioService usuarioService;

    @GetMapping("/{id}")
    public ResponseEntity<UsuarioDTO> obtenerUsuario(@PathVariable Long id) {
        UsuarioDTO usuario = usuarioService.obtenerPorId(id);
        return ResponseEntity.ok(usuario);
    }

    @PostMapping
    public ResponseEntity<UsuarioDTO> crearUsuario(@Valid @RequestBody CrearUsuarioRequest request) {
        UsuarioDTO creado = usuarioService.crear(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(creado);
    }
}
```

## spring-webflux: programacion reactiva

`spring-webflux` es la alternativa reactiva a `spring-webmvc`. Mientras que Spring MVC opera sobre Servlet API (un thread por peticion), Spring WebFlux opera sobre la API de Reactive Streams y puede ejecutarse sobre servidores no bloqueantes como Netty. Esto lo hace adecuado para aplicaciones con alta concurrencia donde el bloqueo de threads es inaceptable.

WebFlux soporta dos modelos de programacion: anotaciones (similar a MVC) y functional endpoints (router functions). Internamente utiliza Project Reactor como implementacion de Reactive Streams, con los tipos `Mono` (un elemento o vacio) y `Flux` (0..N elementos).

```java
@RestController
@RequestMapping("/api/eventos")
public class EventoController {

    @GetMapping
    public Flux<EventoDTO> listarEventos() {
        return eventoService.listarTodos(); // Flux emite eventos de forma reactiva
    }

    @GetMapping("/{id}")
    public Mono<ResponseEntity<EventoDTO>> obtenerEvento(@PathVariable String id) {
        return eventoService.obtenerPorId(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
}
```

## spring-test: testing integrado

`spring-test` proporciona herramientas para probar aplicaciones Spring con soporte para pruebas unitarias, de integracion y de capa completa. `@SpringBootTest` carga el `ApplicationContext` completo para pruebas de integracion, mientras que `@WebMvcTest` carga solo la capa web para pruebas de controllers. `@MockBean` permite inyectar mocks en el contexto de Spring, reemplazando beans reales durante la ejecucion de pruebas.

```java
@WebMvcTest(UsuarioController.class)
class UsuarioControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UsuarioService usuarioService;

    @Test
    void obtenerUsuario_cuandoExiste_retornaOk() throws Exception {
        UsuarioDTO mockUsuario = new UsuarioDTO(1L, "Juan", "juan@email.com");
        when(usuarioService.obtenerPorId(1L)).thenReturn(mockUsuario);

        mockMvc.perform(get("/api/usuarios/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.nombre").value("Juan"));
    }
}
```

## Relacion entre modulos

Los modulos de Spring no operan de forma aislada. Existe una dependencia jerarquica clara: `spring-core` es la base, `spring-beans` depende de `spring-core`, `spring-context` depende de `spring-beans`, y los modulos de mayor nivel (`spring-webmvc`, `spring-tx`, `spring-data`) dependen de `spring-context`. Esta jerarquia refleja la arquitectura en capas del framework: cada capa anade funcionalidad sin modificar la inferior.

En la practica, la mayoria de las aplicaciones modernas con Spring Boot no necesitan declarar dependencias individuales de estos modulos. Los starter dependencies de Spring Boot agrupan los modulos relevantes automaticamente. Sin embargo, comprender esta estructura interna es critico para resolver problemas de configuracion, entender el comportamiento del contenedor y tomar decisiones de arquitectura informadas.

