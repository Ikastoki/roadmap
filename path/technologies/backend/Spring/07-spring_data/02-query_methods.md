# Query Methods

Spring Data permite definir consultas de forma declarativa analizando el nombre de los métodos en la interfaz del repositorio. Este mecanismo, conocido como query derivation, elimina la necesidad de escribir sentencias SQL o JPQL para la mayoría de casos comunes, transformando automáticamente el nombre del método en una consulta ejecutable.

## Derivacion de consultas

El motor de Spring Data parsea el nombre del método y lo convierte en una consulta JPQL equivalente. La nomenclatura sigue un patrón consistente: prefijo del método + nombre de la propiedad + operador lógico opcional.

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    // Buscar por propiedad exacta
    Producto findByNombre(String nombre);

    // Condiciones AND
    List<Producto> findByNombreAndCategoria(String nombre, String categoria);

    // Condiciones OR
    List<Producto> findByNombreOrDescripcion(String nombre, String descripcion);

    // Contiene (LIKE %valor%)
    List<Producto> findByNombreContaining(String termino);

    // Mayor que
    List<Producto> findByPrecioGreaterThan(BigDecimal precio);

    // Menor o igual que
    List<Producto> findByPrecioLessThanEqual(BigDecimal precio);

    // Entre rangos
    List<Producto> findByPrecioBetween(BigDecimal min, BigDecimal max);

    // Is null / Is not null
    List<Producto> findByDescripcionIsNull();
    List<Producto> findByDescripcionIsNotNull();

    // Ordering
    List<Producto> findByOrderByNombreAsc();
    List<Producto> findByOrderByPrecioDesc();

    // Booleanos
    List<Producto> findByActivoTrue();
    List<Producto> findByDescontinuadoFalse();

    // In
    List<Producto> findByCategoriaIn(List<String> categorias);

    // Like manual
    List<Producto> findByNombreLike(String patron);
}
```

> Los prefijos válidos incluyen `find...By`, `read...By`, `query...By`, `count...By`, `exists...By` y `delete...By`. El prefijo `get...By` también es aceptado pero no es recomendado por ambiguiedad con getters.

```java
// Ejemplo: countByCategoria retorna el numero de productos por categoria
long total = productoRepository.countByCategoria("Electronica");

// existsBy retorna boolean
boolean existe = productoRepository.existsByNombre("Laptop Pro");

// deleteBy elimina y retorna el numero de eliminados
@Modifying
@Query("DELETE FROM Producto p WHERE p.activo = false")
int deleteByActivoFalse();
```

## Anotacion @Query

Cuando la derivacion de nombres no es suficiente, `@Query` permite definir consultas JPQL o nativas directamente en el método. JPQL opera sobre entidades y sus propiedades, mientras que las consultas nativas operan sobre tablas y columnas directamente.

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    // Consulta JPQL
    @Query("SELECT p FROM Producto p WHERE p.precio > :precio AND p.activo = true")
    List<Producto> buscarPorPrecioMayor(@Param("precio") BigDecimal precio);

    // Consulta nativa
    @Query(value = "SELECT * FROM productos WHERE categoria = :categoria AND stock > 0",
           nativeQuery = true)
    List<Producto> buscarConStock(@Param("categoria") String categoria);

    // Consulta con ORDER BY en JPQL
    @Query("SELECT p FROM Producto p ORDER BY p.precio DESC")
    Page<Producto> productosOrdenadosPorPrecio(Pageable pageable);
}
```

Las consultas nativas son útiles cuando se necesita acceder a funcionalidades específicas del motor de base de datos que JPQL no expone, como funciones de ventana, CTEs o hints específicos de PostgreSQL o MySQL.

## Modifying queries

Las consultas de modificación (`INSERT`, `UPDATE`, `DELETE`) requieren la anotación `@Modifying` para indicar a Spring Data que no se trata de una consulta SELECT. Además, es necesario configurar la limpieza del contexto de persistencia después de la ejecución.

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    @Modifying
    @Query("UPDATE Producto p SET p.activo = :activo WHERE p.categoria = :categoria")
    int actualizarEstadoPorCategoria(@Param("categoria") String categoria,
                                     @Param("activo") boolean activo);

    @Modifying
    @Query(value = "UPDATE productos SET stock = stock + :cantidad WHERE id = :id",
           nativeQuery = true)
    int incrementarStock(@Param("id") Long id, @Param("cantidad") int cantidad);

    @Modifying
    @Query("DELETE FROM Producto p WHERE p.fechaCreacion < :fecha")
    int eliminarAnterioresA(@Param("fecha") LocalDateTime fecha);
}
```

```java
@Service
@Transactional
public class InventarioService {
    private final ProductoRepository productoRepository;

    public void actualizarInventario(String categoria, int cantidad) {
        // El retorno indica el numero de filas afectadas
        int filas = productoRepository.actualizarEstadoPorCategoria(categoria, true);
        System.out.println("Filas actualizadas: " + filas);
    }
}
```

> Las consultas con `@Modifying` deben ejecutarse dentro de una transacción. Si la consulta se ejecuta fuera de contexto transaccional, Spring lanzará una `TransactionRequiredException`.

## Named Queries

Las named queries se definen a nivel de entidad mediante `@NamedQuery` o en un archivo `META-INF/orm.xml`. Son consultas precompiladas que se reutilizan en múltiples repositorios y se benefician de la validación en tiempo de arranque.

```java
@Entity
@NamedQuery(name = "Producto.findByCategoria",
            query = "SELECT p FROM Producto p WHERE p.categoria = :categoria")
@NamedQuery(name = "Producto.findActivosOrdenadosPorPrecio",
            query = "SELECT p FROM Producto p WHERE p.activo = true ORDER BY p.precio ASC")
public class Producto {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;
    private String categoria;
    private BigDecimal precio;
    private boolean activo;
}
```

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    // Referencia a la named query por nombre
    List<Producto> findByCategoria(@Param("categoria") String categoria);
    List<Producto> findActivosOrdenadosPorPrecio();
}
```

## Consultas con SpEL

Spring Data soporta Spring Expression Language (SpEL) dentro de las consultas `@Query`, lo que permite construir dinámicamente partes de la consulta en tiempo de ejecución.

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    @Query("SELECT p FROM #{#entityName} p WHERE p.precio > :precio")
    List<Producto> buscarPorPrecioDinamico(@Param("precio") BigDecimal precio,
                                           @Param("entityName") String nombreEntidad);
}
```

```java
// Uso dinamico: consulta que puede aplicarse a diferentes entidades
public class BusquedaService {
    private final EntityManager entityManager;

    @PersistenceContext
    public void setEntityManager(EntityManager em) {
        this.entityManager = em;
    }

    public <T> List<T> buscarPorPropiedad(Class<T> clazz, String propiedad, Object valor) {
        String jpql = "SELECT e FROM " + clazz.getSimpleName() + " e WHERE e." + propiedad + " = :valor";
        return entityManager.createQuery(jpql, clazz)
                .setParameter("valor", valor)
                .getResultList();
    }
}
```

La combinación de derivación de nombres, `@Query` y SpEL cubre prácticamente cualquier escenario de consulta, desde los más simples hasta los más complejos, manteniendo el código limpio y tipado.

