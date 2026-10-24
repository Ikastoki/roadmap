# Patron Repository (concepto)

El patron Repository es una abstraccion del almacenamiento de datos que presenta a la capa de negocio una interfaz coherente basada en colecciones de objetos del dominio. En lugar de que el Service interactue con la base de datos mediante queries SQL, APIs de JPA o llamadas a servicios externos, se relaciona con un repositorio que parece una coleccion en memoria de objetos del dominio. Esta abstraccion es uno de los pilares que permite que la capa de negocio permanezca independiente de la infraestructura de persistencia.

## Abstraccion del almacenamiento

La idea central del patron Repository es que la capa de negocio no necesita saber como se almacenan ni de donde provienen los datos. El repositorio actua como un intermediario que traduce las operaciones del dominio (guardar un usuario, buscar un producto) en operaciones de persistencia concretas (INSERT, SELECT, llamada HTTP a un API externa).

```java
// La capa de negocio conoce solo la interfaz
public interface UserRepository {
    Optional<User> findById(UserId id);
    List<User> findAll();
    User save(User user);
    void delete(UserId id);
    boolean existsByEmail(Email email);
}

// El Service usa el repositorio sin saber como se implementa
public class UserService {
    private final UserRepository userRepository;

    public User createUser(String name, Email email) {
        if (userRepository.existsByEmail(email)) {
            throw new DuplicateEmailException(email);
        }
        User user = new User(UserId.generate(), name, email);
        return userRepository.save(user);
    }
}
```

La implementacion concreta del repositorio se encuentra en la capa de persistencia y puede cambiar sin afectar al negocio. Una aplicacion puede comenzar con un `InMemoryUserRepository` para desarrollo y pruebas, y migrar a un `JpaUserRepository` o un `MongoUserRepository` en produccion sin modificar una sola linea del Service.

```java
public class InMemoryUserRepository implements UserRepository {
    private final Map<UserId, User> store = new ConcurrentHashMap<>();

    @Override
    public Optional<User> findById(UserId id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<User> findAll() {
        return List.copyOf(store.values());
    }

    @Override
    public User save(User user) {
        store.put(user.getId(), user);
        return user;
    }

    @Override
    public void delete(UserId id) {
        store.remove(id);
    }

    @Override
    public boolean existsByEmail(Email email) {
        return store.values().stream()
            .anyMatch(u -> u.getEmail().equals(email));
    }
}
```

## Interfaz CRUD

La mayoria de los repositorios siguen un conjunto comun de operaciones de creacion, lectura, actualizacion y eliminacion. Estas operaciones forman el contrato minimo que cualquier repositorio debe cumplir.

```java
public interface CrudRepository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    List<T> findAllById(Iterable<ID> ids);
    T save(T entity);
    List<T> saveAll(Iterable<T> entities);
    void deleteById(ID id);
    void delete(T entity);
    void deleteAll();
    boolean existsById(ID id);
    long count();
}
```

Un repositorio puede extender este contrato basico con metodos de busqueda especificos del dominio. Por ejemplo, un `OrderRepository` puede incluir metodos para buscar pedidos por estado, por cliente o dentro de un rango de fechas.

```java
public interface OrderRepository extends CrudRepository<Order, OrderId> {
    List<Order> findByCustomerId(CustomerId customerId);
    List<Order> findByStatus(OrderStatus status);
    List<Order> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
    long countByStatus(OrderStatus status);
}
```

## Repository generico

Cuando una aplicacion tiene multiples entidades, escribir la misma interfaz CRUD para cada una resulta repetitivo. El repository generico resuelve este problema parametrizando las operaciones basicas con el tipo de la entidad y el tipo de su identificador.

```java
public interface GenericRepository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    Page<T> findAll(Pageable pageable);
    T save(T entity);
    List<T> saveAll(List<T> entities);
    void deleteById(ID id);
    boolean existsById(ID id);
    long count();
}

// Uso concreto
public interface UserRepository extends GenericRepository<User, UserId> {
    Optional<User> findByEmail(Email email);
}

public interface ProductRepository extends GenericRepository<Product, ProductId> {
    List<Product> findByNameContaining(String name);
    Page<Product> findByCategory(Category category, Pageable pageable);
}
```

La implementacion generica puede definirse una unica vez y reutilizarse para todas las entidades. En Spring Data JPA, por ejemplo, `JpaRepository` ya proporciona esta implementacion generica, y los repositorios especificos solo necesitan declarar los metodos de busqueda personalizados.

> El patron Repository no debe confundirse con el patron DAO (Data Access Object). Mientras que el DAO abstrae el acceso a una tabla concreta y suele estar mas cerca del SQL, el Repository abstrae una coleccion de objetos del dominio y esta mas cerca del modelo de negocio. En la practica, la linea es difusa, pero la intencion es diferente: el DAO pertenece a la capa de persistencia; el Repository, al dominio.

## Beneficios del patron Repository

La principal ventaja del patron Repository es el desacoplamiento. Al depender de una interfaz en lugar de una implementacion concreta, la capa de negocio puede cambiar de tecnologia de persistencia sin modificaciones. Esto es util no solo para cambiar de base de datos, sino para probar la logica de negocio sin necesidad de una base de datos real: basta con proporcionar una implementacion en memoria o un mock.

Ademas, el Repository centraliza la logica de acceso a datos. Si se necesita anadir cache a las consultas, registrarlas, auditarlas o aplicar politicas de seguridad, se hace en la implementacion del repositorio, no en los Services. Esto evita dispersion y facilita el mantenimiento.

```java
public class CachedUserRepository implements UserRepository {
    private final UserRepository delegate;
    private final Cache<UserId, User> cache;

    public CachedUserRepository(UserRepository delegate, Cache<UserId, User> cache) {
        this.delegate = delegate;
        this.cache = cache;
    }

    @Override
    public Optional<User> findById(UserId id) {
        User cached = cache.get(id);
        if (cached != null) {
            return Optional.of(cached);
        }
        Optional<User> result = delegate.findById(id);
        result.ifPresent(user -> cache.put(id, user));
        return result;
    }

    @Override
    public User save(User user) {
        cache.put(user.getId(), user);
        return delegate.save(user);
    }
}
```

Este ejemplo muestra un repositorio que envuelve a otro para anadir cache, siguiendo el patron Decorator. El Service que usa `UserRepository` no necesita cambiar: recibe la implementacion cacheada y funciona igual.

> El patron Repository es especialmente valioso en aplicaciones con pruebas automatizadas. Poder cambiar la implementacion del repositorio por una version en memoria permite ejecutar los tests de la capa de negocio en milisegundos, sin configurar bases de datos ni esperar conexiones.
