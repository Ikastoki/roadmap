# Testing de repositorios

Los repositorios son el punto donde la aplicacion Java se encuentra con la base de datos. Probar esta capa implica verificar que las operaciones CRUD funcionan, que las queries JPQL devuelven los resultados esperados, que el mapeo objeto-relacional es correcto y que las restricciones de integridad se aplican. No se trata de probar JPA o Hibernate --eso lo hace su equipo-- sino de probar que el comportamiento definido en el repositorio, especialmente las queries personalizadas y las relaciones entre entidades, se comporta como se espera.

El desafio es hacerlo de forma repetible y sin contaminar entornos compartidos. Dos estrategias resuelven este problema: usar una base de datos en memoria como H2, o usar contenedores Docker con Testcontainers para levantar la misma base de datos que en produccion.

## H2 en memoria

H2 es una base de datos relacional escrita en Java que puede ejecutarse completamente en memoria. Se integra con Spring Boot a traves de `@AutoConfigureTestDatabase` o configurando manualmente el datasource para que apunte a una base H2 efimera.

```yaml
# application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
```

Spring Boot simplifica aun mas: cuando detecta H2 en el classpath y no hay un datasource configurado, levanta automaticamente una base de datos en memoria para los tests. La anotacion `@AutoConfigureTestDatabase(replace = Replace.ANY)` fuerza este comportamiento incluso cuando existe una configuracion de base de datos real.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindUserByEmail() {
        UserEntity user = new UserEntity("Ana", "ana@example.com");
        userRepository.save(user);

        Optional<UserEntity> found = userRepository.findByEmail("ana@example.com");
        assertTrue(found.isPresent());
    }

    @Test
    void shouldReturnEmptyWhenEmailNotFound() {
        Optional<UserEntity> found = userRepository.findByEmail("no-existe@example.com");
        assertFalse(found.isPresent());
    }

    @Test
    void shouldRespectUniqueConstraint() {
        userRepository.save(new UserEntity("Ana", "ana@example.com"));
        assertThrows(DataIntegrityViolationException.class, () -> {
            userRepository.save(new UserEntity("Otro", "ana@example.com"));
            userRepository.flush();
        });
    }
}
```

> `@DataJpaTest` configura solo los beans de JPA (EntityManager, DataSource, repositorios), sin cargar el resto de la aplicacion. Esto hace que los tests de repositorios sean mucho mas rapidos que un `@SpringBootTest`.

La principal limitacion de H2 es que no es identico a la base de datos de produccion. Cada motor tiene su dialecto SQL, sus comportamientos de bloqueo, su manejo de transacciones y sus tipos de datos. Una query que funciona en H2 puede fallar en PostgreSQL por diferencias en la sintaxis de funciones nativas. Para los casos simples --CRUD, queries JPQL estandar-- H2 es suficiente. Para queries nativas o caracteristicas especificas del motor, se necesita la base de datos real.

## Testcontainers

Testcontainers es una libreria Java que gestiona contenedores Docker desde los tests. Permite levantar una instancia de PostgreSQL, MySQL o cualquier otra base de datos en un contenedor efimero, ejecutar los tests contra ella y destruir el contenedor al finalizar.

```java
@Testcontainers
@SpringBootTest
class UserRepositoryTestcontainersTest {

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
    private UserRepository userRepository;

    @Test
    void shouldSaveAndFindUserWithNativeQuery() {
        UserEntity user = new UserEntity("Ana", "ana@example.com");
        userRepository.save(user);

        List<UserEntity> result = userRepository.findByNativeQuery("ana@example.com");
        assertEquals(1, result.size());
        assertEquals("Ana", result.get(0).getName());
    }
}
```

Testcontainers resuelve el problema de la fidelidad: los tests se ejecutan contra la misma base de datos que produccion, con el mismo dialecto SQL, los mismos tipos de datos y el mismo comportamiento transaccional. El coste es el tiempo de arranque del contenedor (unos segundos al inicio de la suite) y la dependencia de Docker en el entorno de ejecucion.

> En CI, Testcontainers se integra con servicios como GitHub Actions, Jenkins o GitLab CI sin configuracion adicional, siempre que Docker esté disponible en el runner.

## Assertions de persistencia

Verificar que los datos se persisten correctamente va mas alla de comprobar que un `save()` no lanza excepcion. Hay que asegurar que el estado guardado es el esperado, que las relaciones se persisten en cascada cuando corresponde y que las restricciones de integridad se aplican.

```java
@Test
void shouldPersistUserWithOrders() {
    UserEntity user = new UserEntity("Ana", "ana@example.com");
    OrderEntity order = new OrderEntity("PROD-001", BigDecimal.valueOf(100));
    user.addOrder(order);
    userRepository.save(user);
    entityManager.flush();

    UserEntity found = entityManager.find(UserEntity.class, user.getId());
    assertNotNull(found);
    assertEquals(1, found.getOrders().size());
    assertEquals("PROD-001", found.getOrders().get(0).getProductCode());
}
```

El uso de `entityManager.flush()` y `entityManager.clear()` es una practica habitual en tests de persistencia. Forzar el flush garantiza que Hibernate envía las sentencias SQL a la base de datos antes de la asercion, evitando falsos positivos donde el objeto todavia esta en la cache de primer nivel y no se ha persistido realmente.

```java
@Test
void shouldApplyUniqueConstraintAtDatabaseLevel() {
    userRepository.save(new UserEntity("Ana", "ana@example.com"));
    userRepository.flush();

    assertThrows(DataIntegrityViolationException.class, () -> {
        userRepository.save(new UserEntity("Otro", "ana@example.com"));
        userRepository.flush();
    });
}
```

La eleccion entre H2 y Testcontainers no es binaria. Muchos proyectos usan H2 para la mayoria de tests de repositorios por su velocidad, y reservan Testcontainers para un subconjunto de tests que ejercitan queries nativas, funciones especificas del motor o comportamientos transaccionales que solo tienen sentido en la base de datos real.
