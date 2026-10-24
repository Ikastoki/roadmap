# Repository generico

El patron Repository se presenta como una abstraccion sobre el almacenamiento persistente: trata cada entidad del dominio como si fuera una coleccion en memoria, independientemente de si los datos residen en una base de datos relacional, un almacen documental, un servicio externo o un simple mapa en memoria. Esta indireccion permite que la capa de negocio manipule objetos de dominio sin acoplarse a los detalles de infraestructura. Una interfaz generica `GenericRepository<T, ID>` parametrizada por el tipo de entidad y el tipo de su identificador establece el contrato basico que toda implementacion debe satisfacer, y a partir de ahi cada entidad concreta especializa dicho contrato con sus propias necesidades de consulta.

## Interfaz generica

La interfaz `GenericRepository<T, ID>` define las operaciones fundamentales de persistencia que cualquier repositorio debe ofrecer. La parametrizacion con tipos genericos permite reutilizar la misma interfaz para todas las entidades del sistema sin duplicar codigo:

```java
public interface GenericRepository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    T save(T entity);
    List<T> saveAll(List<T> entities);
    void deleteById(ID id);
    boolean existsById(ID id);
    long count();
}
```

Cada metodo captura una operacion atomica de acceso a datos:

- `findById`: recupera una entidad por su identificador. Devuelve `Optional` para expresar explicitamente la posibilidad de ausencia, evitando valores nulos y forzando al cliente a considerar ambos escenarios.
- `findAll`: retorna todas las entidades del tipo. En produccion suele acompanarse de paginacion para evitar saturar la memoria con colecciones grandes, pero la interfaz base mantiene la version simple y deja la sobrecarga paginada a la especializacion.
- `save`: persiste una entidad nueva o actualiza una existente. La implementacion determina si la entidad es nueva (insert) o existente (update) basandose en el valor del identificador.
- `saveAll`: persiste multiples entidades en una sola operacion, optimizando el numero de viajes a la base de datos. La implementacion tipica utiliza batch inserts o prepared statements agrupados.
- `deleteById`: elimina la entidad con el identificador dado. Lanza una excepcion si el identificador no existe o, segun la implementacion, simplemente no hace nada.
- `existsById`: verifica la existencia sin cargar la entidad completa, util para validaciones previas sin el coste de una lectura completa.
- `count`: retorna el numero total de entidades.

> La eleccion entre devolver `T` o `Optional<T>` en `findById` tiene consecuencias en la API. `Optional` comunica al cliente que el valor puede no estar presente y le obliga a tratarlo, mientras que devolver `null` o lanzar una excepcion desplaza esa responsabilidad a la documentacion o al runtime.

Para escenarios que requieran paginacion y ordenacion, la interfaz se extiende con sobrecargas adicionales:

```java
public interface PagingAndSortingRepository<T, ID> extends GenericRepository<T, ID> {
    List<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}
```

`Page` encapsula no solo los resultados de la pagina actual, sino tambien metadatos como el numero total de elementos, el numero de paginas disponibles y la pagina solicitada, informacion que los clientes suelen necesitar para construir controles de navegacion.

## Implementacion generica

Una vez definido el contrato, cada tecnologia de persistencia proporciona su propia implementacion. La clave del patron es que el codigo de negocio depende unicamente de la interfaz, no de la implementacion concreta. Esto permite intercambiar el mecanismo de almacenamiento sin modificar una sola linea de la logica de negocio.

### Implementacion con JPA

JPA (Java Persistence API) es el estandar para acceso a bases de datos relacionales en Java. Una implementacion tipica utiliza `EntityManager`, el nucleo del contexto de persistencia JPA, para traducir cada operacion de la interfaz a su equivalente en SQL:

```java
public class JpaGenericRepository<T, ID> implements GenericRepository<T, ID> {
    private final EntityManager entityManager;
    private final Class<T> entityClass;

    public JpaGenericRepository(EntityManager entityManager, Class<T> entityClass) {
        this.entityManager = entityManager;
        this.entityClass = entityClass;
    }

    @Override
    public Optional<T> findById(ID id) {
        return Optional.ofNullable(entityManager.find(entityClass, id));
    }

    @Override
    public List<T> findAll() {
        CriteriaQuery<T> query = entityManager.getCriteriaBuilder()
                .createQuery(entityClass);
        query.select(query.from(entityClass));
        return entityManager.createQuery(query).getResultList();
    }

    @Override
    @Transactional
    public T save(T entity) {
        if (getId(entity) == null) {
            entityManager.persist(entity);
            return entity;
        } else {
            return entityManager.merge(entity);
        }
    }

    @Override
    @Transactional
    public List<T> saveAll(List<T> entities) {
        List<T> results = new ArrayList<>();
        for (T entity : entities) {
            results.add(save(entity));
        }
        entityManager.flush();
        return results;
    }

    @Override
    @Transactional
    public void deleteById(ID id) {
        findById(id).ifPresent(entityManager::remove);
    }

    @Override
    public boolean existsById(ID id) {
        return findById(id).isPresent();
    }

    @Override
    public long count() {
        CriteriaQuery<Long> query = entityManager.getCriteriaBuilder()
                .createQuery(Long.class);
        query.select(entityManager.getCriteriaBuilder()
                .count(query.from(entityClass)));
        return entityManager.createQuery(query).getSingleResult();
    }
}
```

El metodo `save` distingue entre entidades nuevas y existentes mediante el valor del identificador. Si es nulo, asume que es una nueva entidad y llama a `persist`; si no, llama a `merge`, que copia el estado de la entidad desadjuntada al contexto de persistencia actual. Esta distincion es crucial porque JPA mantiene un contexto de persistencia donde cada entidad solo puede estar asociada a una unica instancia en memoria.

En aplicaciones Spring Boot, raramente es necesario escribir esta implementacion a mano. `JpaRepository<T, ID>` de Spring Data JPA proporciona exactamente esta funcionalidad de forma automatica, implementando metodos como `findById`, `save` o `findAll` mediante la generacion de consultas en tiempo de ejecucion:

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Sin embargo, comprender la implementacion subyacente resulta esencial para diagnosticar problemas de rendimiento, como el n+1 queries, o para decidir cuando usar `save` frente a `saveAndFlush`.

### Implementacion con JDBC

Cuando no se desea la sobrecarga de JPA o se necesita control directo sobre el SQL, se implementa el repositorio con JDBC plano o con plantillas como `JdbcTemplate` de Spring:

```java
public class JdbcGenericRepository<T, ID> implements GenericRepository<T, ID> {
    private final JdbcTemplate jdbcTemplate;
    private final RowMapper<T> rowMapper;
    private final String tableName;
    private final String idColumn;
    private final Function<T, ID> idExtractor;

    @Override
    public Optional<T> findById(ID id) {
        String sql = "SELECT * FROM " + tableName + " WHERE " + idColumn + " = ?";
        List<T> results = jdbcTemplate.query(sql, rowMapper, id);
        return results.stream().findFirst();
    }

    @Override
    public List<T> findAll() {
        String sql = "SELECT * FROM " + tableName;
        return jdbcTemplate.query(sql, rowMapper);
    }

    @Override
    public T save(T entity) {
        if (idExtractor.apply(entity) == null) {
            return insert(entity);
        } else {
            return update(entity);
        }
    }
    // ...
}
```

La implementacion JDBC resulta mas verbosa porque cada operacion requiere escribir SQL explicito, mapear manualmente las filas a objetos y gestionar las conexiones. No obstante, ofrece un rendimiento predecible y total transparencia sobre las consultas que se ejecutan, ventajas que pesan en sistemas con requisitos de latencia extremos.

### Implementacion con NoSQL

Para bases de datos NoSQL, la implementacion varia segun el modelo de datos subyacente. En un almacen documental como MongoDB, las operaciones se traducen a documentos JSON:

```java
public class MongoGenericRepository<T, ID> implements GenericRepository<T, ID> {
    private final MongoCollection<Document> collection;
    private final Class<T> entityClass;

    @Override
    public Optional<T> findById(ID id) {
        Document doc = collection.find(Filters.eq("_id", id)).first();
        return Optional.ofNullable(doc).map(d -> deserialize(d, entityClass));
    }

    @Override
    public T save(T entity) {
        Document doc = serialize(entity);
        if (getId(entity) == null) {
            collection.insertOne(doc);
        } else {
            collection.replaceOne(Filters.eq("_id", getId(entity)), doc);
        }
        return entity;
    }
    // ...
}
```

Cada implementacion comparte la misma interfaz, pero los detalles internos difieren radicalmente: mientras JPA gestiona un contexto de persistencia con cache de primer nivel y deteccion automatica de cambios, JDBC ejecuta SQL directo sin estado intermedio, y MongoDB opera sobre documentos JSON sin esquema fijo. La abstraccion del repositorio generico oculta estas diferencias.

## Especializacion por entidad

La interfaz generica proporciona las operaciones basicas, pero cada entidad del dominio suele requerir consultas especificas que no tienen cabida en un repositorio universal. La especializacion consiste en crear una interfaz por entidad que extienda `GenericRepository` y anada metodos de busqueda propios de ese tipo:

```java
public interface UserRepository extends GenericRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByActiveTrue();
    boolean existsByEmail(String email);
    Page<User> findByNameContaining(String name, Pageable pageable);
}

public interface OrderRepository extends GenericRepository<Order, Long> {
    List<Order> findByUserId(Long userId);
    List<Order> findByStatusAndCreatedAtAfter(OrderStatus status, LocalDateTime date);
    long countByStatus(OrderStatus status);
}
```

Cada repositorio especializado expone las consultas que la capa de negocio necesita, y solo esas. Esto mantiene la interfaz cohesiva: un repositorio de usuarios no ofrece metodos para buscar pedidos, y viceversa.

En Spring Data JPA, los mecanismos de query derivation permiten definir estas consultas simplemente declarando el metodo con el nombre adecuado. El framework analiza el nombre del metodo, extrae las clausulas de busqueda y genera automaticamente la consulta JPQL o SQL correspondiente:

- `findByEmail` genera `WHERE email = ?1`
- `findByNameContaining` genera `WHERE name LIKE %?1%`
- `countByStatus` genera `SELECT COUNT(*) WHERE status = ?1`

Para consultas que no pueden expresarse mediante la convencion de nombres, se utiliza la anotacion `@Query` con JPQL o SQL nativo:

```java
public interface UserRepository extends GenericRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
    Optional<User> findActiveByEmail(@Param("email") String email);

    @Query(value = "SELECT * FROM users WHERE created_at >= :since", nativeQuery = true)
    List<User> findRecentUsers(@Param("since") LocalDateTime since);
}
```

> La especializacion no debe replicar metodos que ya cubre la interfaz generica. Si todas las entidades necesitan `findById`, este metodo vive en `GenericRepository`. Si solo ciertas entidades requieren busqueda por email, ese metodo vive en el repositorio especializado de esa entidad.
