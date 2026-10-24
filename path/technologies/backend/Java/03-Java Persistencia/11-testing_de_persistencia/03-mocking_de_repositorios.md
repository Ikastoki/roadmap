# Mocking de repositorios

El mocking de repositorios es una tecnica fundamental para aislar la capa de servicio de la capa de persistencia durante las pruebas unitarias. En lugar de ejecutar consultas reales contra una base de datos, se sustituyen los repositorios por objetos simulados que devuelven comportamientos predefinidos. Spring Boot integra Mockito de forma nativa, proporcionando la anotacion `@MockBean` para reemplazar beans reales por mocks en el contexto de Spring.

## @MockBean

`@MockBean` crea un mock de Mockito y lo registra en el contexto de aplicacion de Spring, reemplazando cualquier bean existente del mismo tipo. Se utiliza en tests que cargan el contexto de Spring (`@SpringBootTest`, `@WebMvcTest`, etc.) para aislar componentes especificos.

```java
@SpringBootTest
class UsuarioServiceTest {

    @MockBean
    private UsuarioRepository usuarioRepository;

    @Autowired
    private UsuarioService usuarioService;

    @Test
    void deberiaDevolverUsuarioCuandoExiste() {
        Usuario usuarioEsperado = new Usuario("juan@email.com", "Juan", "Perez");
        usuarioEsperado.setId(1L);

        when(usuarioRepository.findById(1L))
            .thenReturn(Optional.of(usuarioEsperado));

        Usuario resultado = usuarioService.obtenerPorId(1L);

        assertThat(resultado).isNotNull();
        assertThat(resultado.getEmail()).isEqualTo("juan@email.com");
    }
}
```

`@MockBean` se puede aplicar tanto a nivel de campo como de metodo. A nivel de metodo es util cuando se necesita un comportamiento diferente para distintos tests:

```java
@SpringBootTest
class UsuarioServiceTest {

    @Autowired
    private UsuarioService usuarioService;

    private UsuarioRepository usuarioRepository;

    @BeforeEach
    void setUp() {
        usuarioRepository = mock(UsuarioRepository.class);
        usuarioService = new UsuarioService(usuarioRepository);
    }
}
```

## Configuracion de comportamientos con when().thenReturn()

Mockito proporciona una API fluida para definir el comportamiento de los metodos del mock. `when()` especifica la llamada a simular y `thenReturn()` define el valor devuelto:

```java
when(usuarioRepository.findById(1L)).thenReturn(Optional.of(usuario));
when(usuarioRepository.findByEmail("test@email.com")).thenReturn(Optional.of(usuario));
when(usuarioRepository.findAll()).thenReturn(List.of(usuario1, usuario2));
```

Para comportamientos condicionales, se pueden encadenar respuestas:

```java
when(usuarioRepository.findById(1L))
    .thenReturn(Optional.of(usuario))
    .thenReturn(Optional.empty());
```

La primera llamada devuelve el usuario, la segunda devuelve vacio.

Para lanzar excepciones:

```java
when(usuarioRepository.findById(999L)).thenThrow(new IllegalArgumentException("ID invalido"));
```

## Verificacion con verify()

`verify()` comprueba que se haya realizado una llamada especifica al mock, con los parametros esperados y el numero de veces deseado:

```java
@Test
void deberiaLlamarABuscarPorId() {
    when(usuarioRepository.findById(1L)).thenReturn(Optional.of(new Usuario()));

    usuarioService.obtenerPorId(1L);

    verify(usuarioRepository).findById(1L);
}
```

`verify()` permite verificar:
- Numero exacto de invocaciones: `verify(mock, times(2)).metodo()`
- Al menos una vez: `verify(mock, atLeastOnce()).metodo()`
- Nunca: `verify(mock, never()).metodo()`
- Al menos N veces: `verify(mock, atLeast(3)).metodo()`
- Maximo N veces: `verify(mock, atMost(2)).metodo()`

```java
@Test
void noDeberiaLlamarAEliminarCuandoNoExiste() {
    when(usuarioRepository.findById(999L)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> usuarioService.eliminar(999L))
        .isInstanceOf(UsuarioNoEncontradoException.class);

    verify(usuarioRepository, never()).delete(any());
}
```

## Argument matchers

Mockito proporciona matchers para parametros cuando no se conoce el valor exacto en tiempo de escritura del test:

```java
when(usuarioRepository.findById(anyLong())).thenReturn(Optional.of(usuario));
when(usuarioRepository.save(any(Usuario.class))).thenReturn(usuario);
verify(usuarioRepository).save(argThat(u -> u.getEmail().contains("@")));
```

`anyLong()`, `anyString()`, `any(Clase.class)` coinciden con cualquier valor del tipo especificado. `argThat()` permite validar condiciones personalizadas.

> Los matchers no deben mezclarse con valores literales en la misma llamada. Si un parametro usa matcher, todos los parametros deben usar matchers o envolver los literales con `eq()`.

## @ExtendWith(MockitoExtension.class)

Para tests que no necesitan el contexto completo de Spring, se puede usar `MockitoExtension` directamente con JUnit 5:

```java
@ExtendWith(MockitoExtension.class)
class UsuarioServiceUnitTest {

    @Mock
    private UsuarioRepository usuarioRepository;

    @InjectMocks
    private UsuarioService usuarioService;

    @Test
    void deberiaDevolverUsuarioCuandoExiste() {
        when(usuarioRepository.findById(1L))
            .thenReturn(Optional.of(new Usuario("test@email.com", "Test", "User")));

        Usuario resultado = usuarioService.obtenerPorId(1L);

        assertThat(resultado).isNotNull();
    }
}
```

`@Mock` crea el mock, `@InjectMocks` crea una instancia del servicio e inyecta los mocks en sus dependencias. Este enfoque es mas rapido que `@SpringBootTest` porque no arranca el contenedor de Spring.

## Buenas practicas

El mocking excesivo puede llevar a tests que solo verifican la implementacion en lugar del comportamiento. Se recomienda mockear unicamente las dependencias directas del componente bajo prueba. Para la capa de repositorios, los tests con `@DataJpaTest` y H2 o TestContainers son preferibles cuando se necesita verificar consultas reales. El mocking es adecuado para la capa de servicio, donde el objetivo es validar la logica de negocio y la orquestacion de llamadas al repositorio.
