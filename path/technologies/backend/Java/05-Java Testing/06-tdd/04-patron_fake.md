# Patron Fake

Cuando una unidad de codigo depende de componentes externos (bases de datos, APIs HTTP, sistemas de ficheros), los tests unitarios no deben usar esas implementaciones reales porque introducen lentitud, fragilidad y dependencias no controladas. Los test doubles resuelven este problema, y dentro de ellos el patron Fake ocupa un lugar destacado.

## TDD con mocks

En TDD, los doubles de prueba se integran en el ciclo normal. Durante la fase Red se disena la interaccion entre la unidad bajo test y sus dependencias, definiendo como se comportara el double para que el test pueda validar la logica deseada.

La pregunta clave en TDD con doubles no es "como implemento esto", sino "como se comunica mi unidad con sus dependencias". Esta perspectiva, conocida como outside-in, prioriza el contrato entre componentes sobre los detalles internos.

## Patron Fake vs Mocks

Existen varios tipos de test doubles, siendo los mas relevantes:

- **Fake**: Implementacion simplificada y funcional de una dependencia que funciona en memoria. Tiene comportamiento real pero sin efectos secundarios (por ejemplo, un `FakeUserRepository` que guarda usuarios en un `HashMap`).
- **Mock**: Objeto que registra las llamadas recibidas y permite verificar que se invocaron los metodos esperados. No tiene logica de negocio; solo expectativas.
- **Stub**: Objeto que devuelve valores prefijados para las llamadas que recibe. No registra interacciones.
- **Spy**: Envoltorio alrededor de un objeto real que registra las llamadas para su posterior verificacion.

La diferencia fundamental entre Fake y Mock es que el Fake tiene comportamiento real (persiste, consulta, filtra) mientras que el Mock solo verifica interaccion.

> Usa Fake cuando necesites comportamiento simulando la dependencia real. Usa Mock cuando necesites verificar que se invocaron ciertos metodos con ciertos argumentos.

## TDD con doubles

El proceso TDD con doubles sigue el mismo ciclo Red-Green-Refactor, pero la fase de diseno incluye decidir como se comportara el double.

1. Se escribe el test usando el double como dependencia
2. El double aun no existe, asi que el test falla por compilacion (Red)
3. Se implementa el double (Fake, Mock o Stub) con la interfaz requerida
4. Se implementa la unidad bajo test usando el double
5. El test pasa (Green)
6. Se refactoriza tanto el codigo de produccion como el double

## Ejemplo UserService con UserRepository

Supongamos un servicio que gestiona usuarios y depende de un repositorio. Aplicamos TDD con un Fake.

**Paso 1 - Definir la interfaz (puede surgir del test):**

```java
public interface UserRepository {
    User save(User user);
    Optional<User> findById(String id);
}
```

**Paso 2 - Red: escribir el test:**

```java
class UserServiceTest {

    private UserRepository repository;
    private UserService service;

    @BeforeEach
    void setUp() {
        repository = new FakeUserRepository();
        service = new UserService(repository);
    }

    @Test
    void shouldCreateUserWithEncryptedPassword() {
        User user = service.createUser("john", "plain123");

        assertEquals("john", user.username());
        assertNotEquals("plain123", user.password());
        assertTrue(repository.findById(user.id()).isPresent());
    }
}
```

**Paso 3 - Green: implementar el Fake y la logica minima:**

```java
public class FakeUserRepository implements UserRepository {
    private final Map<String, User> store = new HashMap<>();

    @Override
    public User save(User user) {
        store.put(user.id(), user);
        return user;
    }

    @Override
    public Optional<User> findById(String id) {
        return Optional.ofNullable(store.get(id));
    }
}

public class UserService {
    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public User createUser(String username, String password) {
        String encrypted = encrypt(password);
        User user = new User(UUID.randomUUID().toString(), username, encrypted);
        return repository.save(user);
    }
}
```

**Paso 4 - Refactor:**

Se puede introducir un `PasswordEncoder` como dependencia y extraer la generacion de ID a un servicio separado, siempre con la proteccion de los tests existentes.

> El Fake es preferible al Mock cuando la dependencia tiene comportamiento de almacenamiento o consulta. Para interacciones sin estado (envio de emails, logging), los Mocks son mas apropiados.
