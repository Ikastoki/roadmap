# Que es Spring Data

Spring Data es un proyecto del ecosistema Spring que abstrae el acceso a bases de datos mediante una capa de repositorios genéricos, eliminando la necesidad de escribir código repetitivo para operaciones CRUD y consultas. Su filosofía se basa en el patrón Repository, que actúa como una capa de abstracción entre la lógica de negocio y la persistencia, proporcionando una API unificada independient del mecanismo de almacenamiento subyacente.

El núcleo de Spring Data se apoya en interfaces genéricas que encapsulan las operaciones más comunes. Estas interfaces利用an la herencia para ofrecer funcionalidad progresivamente más completa, desde operaciones básicas hasta paginación, ordenación y gestión de transacciones.

## CrudRepository\<T, ID\>

`CrudRepository` es la interfaz base que define las operaciones CRUD esenciales para cualquier entidad persistente. El parámetro `T` representa el tipo de la entidad y `ID` el tipo de su identificador.

```java
public interface CrudRepository<T, ID> {
    <S extends T> S save(S entity);
    Optional<T> findById(ID id);
    Iterable<T> findAll();
    void deleteById(ID id);
    long count();
    boolean existsById(ID id);
    // ... additional methods
}
```

El método `save()` distingue internamente entre inserción y actualización mediante la presencia o ausencia del identificador. Si el ID es `null`, se ejecuta un INSERT; de lo contrario, se ejecuta un UPDATE. Este comportamiento se implementa a través del `Persistable` o del análisis del estado de la entidad.

```java
@Repository
public interface ClienteRepository extends CrudRepository<Cliente, Long> {
}

// Uso
@Service
public class ClienteService {
    private final ClienteRepository repository;

    public ClienteService(ClienteRepository repository) {
        this.repository = repository;
    }

    public Cliente guardar(Cliente cliente) {
        return repository.save(cliente);
    }

    public Optional<Cliente> buscarPorId(Long id) {
        return repository.findById(id);
    }

    public List<Cliente> listarTodos() {
        return (List<Cliente>) repository.findAll();
    }

    public void eliminar(Long id) {
        repository.deleteById(id);
    }
}
```

> Spring Data genera automáticamente una implementación concreta en tiempo de ejecución mediante proxy dinámico. No es necesario escribir implementaciones manuales para repositorios simples.

## PagingAndSortingRepository\<T, ID\>

`PagingAndSortingRepository` extiende `CrudRepository` añadiendo soporte para paginación y ordenación, dos operaciones fundamentales cuando se trabaja con volúmenes significativos de datos.

```java
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}
```

La paginación se implementa mediante `PageRequest`, que encapsula el número de página (empezando en 0), el tamaño de página y opcionalmente un criterio de ordenación. Esto se traduce en una consulta SQL con `LIMIT` y `OFFSET` de forma transparente.

```java
@Repository
public interface ProductoRepository extends PagingAndSortingRepository<Producto, Long> {

    Page<Producto> findAll(Pageable pageable);
}

// Uso con paginación y ordenación
Pageable pageable = PageRequest.of(0, 10, Sort.by("precio").ascending());
Page<Producto> pagina = productoRepository.findAll(pageable);

long totalElementos = pagina.getTotalElements();
int totalPaginas = pagina.getTotalPages();
List<Producto> productos = pagina.getContent();
```

## JpaRepository\<T, ID\>

`JpaRepository` es la interfaz más completa para persistencia JPA. Extiende tanto `PagingAndSortingRepository` como `QueryByExampleExecutor`, añadiendo métodos específicos de JPA que permiten un control más fino sobre la ejecución y el ciclo de vida de las entidades.

```java
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>,
        QueryByExampleExecutor<T> {

    void flush();
    <S extends T> S saveAndFlush(S entity);
    void deleteInBatch();
    void deleteAllInBatch();
    T getOne(ID id);
    <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);
}
```

- `flush()` fuerza la sincronización del contexto de persistencia con la base de datos, enviando todas las sentencias SQL pendientes sin finalizar la transacción.
- `saveAndFlush()` combina `save()` con `flush()`, útil cuando se necesita que los cambios sean visibles inmediatamente para consultas posteriores dentro de la misma transacción.
- `deleteInBatch()` ejecuta un `DELETE` masivo sin cargar las entidades en memoria, a diferencia de `deleteAll()` que las carga una por una.
- `getOne()` devuelve una referencia lazy sin inicializar los campos, lanzando una excepción si la entidad no existe (a diferencia de `findById()` que retorna `Optional`).

```java
@Repository
public interface FacturaRepository extends JpaRepository<Factura, Long> {

    @Query("SELECT f FROM Factura f WHERE f.estado = :estado")
    List<Factura> findByEstado(@Param("estado") EstadoFactura estado);
}

@Service
public class FacturaService {
    private final FacturaRepository facturaRepository;

    public void procesarFactura(Factura factura) {
        Factura guardada = facturaRepository.saveAndFlush(factura);
        facturaRepository.flush(); // Forzar envio a BD
    }

    public void eliminarTodas() {
        facturaRepository.deleteInBatch(); // DELETE masivo sin carga
    }
}
```

> La elección entre `CrudRepository`, `PagingAndSortingRepository` y `JpaRepository` depende del nivel de control necesario. Para la mayoría de aplicaciones, `JpaRepository` ofrece el equilibrio adecuado entre funcionalidad y simplicidad.

## Resumen de herencia

```
Repository (marcador base)
    └── CrudRepository<T, ID> (CRUD basico)
            └── PagingAndSortingRepository<T, ID> (paginacion y ordenacion)
                    └── JpaRepository<T, ID> (JPA especifico)
```

Cada nivel añade funcionalidad sin eliminar la anterior, lo que permite al desarrollador elegir el nivel de abstracción que mejor se adapte a sus necesidades. La infraestructura de Spring Data se encarga de generar la implementación concreta en tiempo de ejecución, analizando los nombres de los métodos y las anotaciones presentes en la interfaz del repositorio.

