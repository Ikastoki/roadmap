# Testing por capa

La arquitectura de capas ofrece una ventaja estrategica para el testing: cada capa puede probarse de forma aislada, con sus propias tecnicas y herramientas, sin necesidad de levantar el sistema completo. Esta testabilidad no es un subproducto casual sino una consecuencia directa de la separacion de responsabilidades y la inversion de dependencias. Cuando un Controller depende de una interfaz de Servicio, y un Servicio depende de una interfaz de Repositorio, cada uno puede sustituir sus dependencias reales por dobles durante las pruebas.

Tres niveles de testing convergen en esta arquitectura: unitario, de integracion y end-to-end. Cada uno opera en un circulo distinto, con velocidades de ejecucion y costes de mantenimiento diferentes. La piramide de testing --unitaria en la base, integracion en el medio, E2E en la cuspide-- sigue siendo la guia para decidir cuantas pruebas de cada tipo escribir.

## Unit tests

Las pruebas unitarias verifican el comportamiento de la unidad mas pequena de software --tipicamente un metodo o una clase-- de forma completamente aislada. En una arquitectura de capas, cualquier clase que tenga dependencias inyectadas por constructor puede someterse a tests unitarios sin necesidad de infraestructura: la dependencia se sustituye por un mock o un stub que devuelve valores prefijados.

```java
public class UserServiceTest {

    private UserRepository userRepository;
    private UserService userService;

    @BeforeEach
    void setUp() {
        userRepository = mock(UserRepository.class);
        userService = new UserService(userRepository);
    }

    @Test
    void shouldCreateUserWhenEmailIsUnique() {
        CreateUserRequest request = new CreateUserRequest("Ana", "ana@example.com");
        when(userRepository.existsByEmail("ana@example.com")).thenReturn(false);
        when(userRepository.save(any(User.class))).thenReturn(new User(1L, "Ana", "ana@example.com"));

        UserDTO result = userService.createUser(request);

        assertNotNull(result);
        assertEquals("Ana", result.name());
        verify(userRepository).save(any(User.class));
    }

    @Test
    void shouldThrowExceptionWhenEmailAlreadyExists() {
        CreateUserRequest request = new CreateUserRequest("Ana", "ana@example.com");
        when(userRepository.existsByEmail("ana@example.com")).thenReturn(true);

        assertThrows(BusinessException.class, () -> userService.createUser(request));
        verify(userRepository, never()).save(any());
    }
}
```

Un test unitario es rapido (milisegundos), determinista (mismo input produce mismo output) y no toca la base de datos, el sistema de archivos ni la red. La mayoria de las pruebas en un proyecto deberian ser unitarias.

## Integration tests

Las pruebas de integracion verifican que dos o mas componentes funcionan correctamente cuando colaboran. En la practica, esto significa probar el Servicio contra un Repositorio real (aunque sea en memoria) o probar el Repositorio contra una base de datos real. Son mas lentas que las unitarias porque requieren levantar contexto, cargar datos y a veces conectar con infraestructura, pero ofrecen una confianza mayor: un test unitario que mockea el Repositorio no detecta errores en la query JPQL.

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class UserRepositoryIntegrationTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndFindUser() {
        UserEntity user = new UserEntity("Ana", "ana@example.com");
        UserEntity saved = userRepository.save(user);

        Optional<UserEntity> found = userRepository.findById(saved.getId());

        assertTrue(found.isPresent());
        assertEquals("ana@example.com", found.get().getEmail());
    }
}
```

Las pruebas de integracion ocupan el nivel medio de la piramide. Cubren los puntos donde las capas se tocan: un Servicio llamando a un Repositorio, un Controller llamando a un Servicio a traves de HTTP.

## End-to-end tests

Las pruebas end-to-end (E2E) ejercitan la aplicacion completa, desde la interfaz externa hasta la base de datos, simulando escenarios de uso real. En una API REST, esto significa lanzar peticiones HTTP reales contra la aplicacion desplegada --o contra un contenedor de la aplicacion-- y verificar que la respuesta y el estado persistido son correctos.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserE2ETest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldCreateUserEndToEnd() {
        CreateUserRequest request = new CreateUserRequest("Ana", "ana@example.com");

        ResponseEntity<UserResponse> response = restTemplate.postForEntity(
            "/api/users", request, UserResponse.class);

        assertEquals(201, response.getStatusCodeValue());
        assertTrue(response.getBody().id() > 0);

        assertTrue(userRepository.findByEmail("ana@example.com").isPresent());
    }
}
```

Las pruebas E2E son lentas (segundos o minutos), fragiles (dependen de toda la infraestructura) y costosas de mantener. Por eso solo deben cubrir los flujos criticos del sistema. Su valor reside en que detectan problemas que los tests unitarios y de integracion no pueden capturar: configuracion incorrecta de la aplicacion, errores en el mapeo de rutas, incompatibilidades entre versiones de librerias.

## Mocking y stubbing

Mocking es la tecnica de sustituir un objeto real por un doble que registra como se le llama y que puede devolver respuestas programadas. Stubbing es el caso particular en que el doble simplemente devuelve un valor fijo cuando se invoca un metodo concreto. Ambos son esenciales para aislar la unidad bajo prueba.

Mockito es la libreria estandar en el ecosistema Java. Proporciona dos operaciones fundamentales: `mock()` para crear el doble y `when().thenReturn()` para definir su comportamiento.

```java
@Test
void shouldVerifyInteraction() {
    UserRepository mockRepo = mock(UserRepository.class);
    when(mockRepo.findById(1L)).thenReturn(Optional.of(new User(1L, "Ana", "ana@example.com")));

    UserService service = new UserService(mockRepo);
    UserDTO result = service.findById(1L);

    assertEquals("Ana", result.name());
    verify(mockRepo).findById(1L);
}
```

> Mockear no es gratis: un mock que no refleja el comportamiento real de la implementacion produce falsos positivos. La regla general es mockear lo que no se posee o lo que es lento (bases de datos, servicios HTTP externos) y no mockear lo que se esta probando.

La combinacion de tests unitarios con mocks, tests de integracion con infraestructura real y tests E2E sobre el sistema completo proporciona una cobertura jerarquica donde los fallos se detectan en el nivel mas cercano a su origen.
