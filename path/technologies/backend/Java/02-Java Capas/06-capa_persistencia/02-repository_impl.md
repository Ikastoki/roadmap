# Repository (implementacion)

El patron Repository separa la logica de acceso a datos de la logica de negocio mediante una interfaz definida en la capa de dominio y una implementacion concreta en la capa de persistencia. La interfaz expresa operaciones en el lenguaje del dominio (`findByEmail`, `findActiveUsers`, `countByRole`), mientras que la implementacion traduce esas operaciones a queries contra la base de datos subyacente. El servicio de negocio depende de la interfaz, nunca de la implementacion, lo que permite cambiar el mecanismo de persistencia sin modificar una sola linea de la logica de negocio.

En JPA, Spring Data JPA proporciona implementaciones automaticas para las interfaces que extienden `JpaRepository`, eliminando la necesidad de escribir codigo de persistencia repetitivo. Sin embargo, el patron sigue siendo valido incluso sin Spring: la interfaz en el dominio y el `EntityManager` en la implementacion manual mantienen el mismo principio de inversion de dependencias.

## CRUD basico

Las operaciones fundamentales de cualquier repositorio son crear, leer, actualizar y eliminar. Spring Data JPA las proporciona a traves de metodos heredados de `JpaRepository`.

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    // Metodos heredados:
    // Usuario save(Usuario entity)
    // Optional<Usuario> findById(Long id)
    // List<Usuario> findAll()
    // void deleteById(Long id)
    // boolean existsById(Long id)
    // long count()
}
```

Sin Spring, la implementacion manual con `EntityManager` tiene la misma estructura pero requiere escribir el codigo de persistencia:

```java
public class UsuarioRepositoryImpl implements UsuarioRepository {
    private final EntityManager em;

    public UsuarioRepositoryImpl(EntityManager em) {
        this.em = em;
    }

    @Override
    public Usuario save(Usuario usuario) {
        if (usuario.getId() == null) {
            em.persist(usuario);
            return usuario;
        } else {
            return em.merge(usuario);
        }
    }

    @Override
    public Optional<Usuario> findById(Long id) {
        return Optional.ofNullable(em.find(Usuario.class, id));
    }

    @Override
    public List<Usuario> findAll() {
        return em.createQuery("SELECT u FROM Usuario u", Usuario.class)
                .getResultList();
    }

    @Override
    public void deleteById(Long id) {
        findById(id).ifPresent(em::remove);
    }
}
```

> La implementacion manual de `save` utiliza `persist` para entidades nuevas (sin ID) y `merge` para entidades existentes. Esta distincion es importante porque `persist` anade la entidad al contexto de persistencia, mientras que `merge` devuelve una copia gestionada de una entidad detached.

## Custom queries con JPQL

JPQL (Java Persistence Query Language) es un lenguaje de consultas orientado a objetos que opera sobre las entidades JPA en lugar de sobre las tablas de la base de datos. Las queries JPQL son independientes del motor de base de datos subyacente.

```java
@Query("SELECT u FROM Usuario u WHERE u.email = :email")
Optional<Usuario> findByEmail(@Param("email") String email);

@Query("SELECT u FROM Usuario u WHERE u.activo = true ORDER BY u.nombre")
List<Usuario> findActiveUsers();

@Query("SELECT u FROM Usuario u WHERE u.edad >= :edadMinima AND u.ciudad = :ciudad")
List<Usuario> findByEdadAndCiudad(@Param("edadMinima") int edad,
                                  @Param("ciudad") String ciudad);
```

Los parametros se vinculan por nombre con `:parametro` y `@Param`. Tambien es posible usar parametros posicionales con `?1`, `?2`, etc., pero la vinculacion por nombre es mas legible y menos propensa a errores al refactorizar.

## Custom queries con Native SQL

Cuando JPQL no es suficiente (funciones de base de datos especificas, consultas complejas de rendimiento critico, tablas no mapeadas como entidades), se puede recurrir a SQL nativo.

```java
@Query(value = "SELECT * FROM usuarios WHERE creado_en > :fecha", nativeQuery = true)
List<Usuario> findRecentUsers(@Param("fecha") LocalDateTime fecha);

@Query(value = "SELECT u.id, u.nombre, COUNT(p.id) as total_pedidos " +
       "FROM usuarios u JOIN pedidos p ON u.id = p.usuario_id " +
       "GROUP BY u.id, u.nombre " +
       "HAVING COUNT(p.id) >= :minPedidos",
       nativeQuery = true)
List<Object[]> findUsersWithMinOrders(@Param("minPedidos") long minPedidos);
```

Las queries nativas devuelven `Object[]` o, si se especifica `resultSetMapping`, entidades mapeadas. El mapeo manual de resultados nativos a DTOs es una practica habitual cuando la estructura de la query difiere significativamente del esquema de las entidades.

> El uso de SQL nativo rompe la abstraccion de JPA y acopla la consulta a un dialecto concreto de base de datos. Solo deberia usarse cuando la diferencia de rendimiento lo justifique o cuando la funcionalidad (consultas recursivas, funciones ventana, tipos geometricos) no este disponible en JPQL.

## Pagination y sorting

La paginacion es indispensable en cualquier aplicacion que maneje conjuntos de datos medianos o grandes. Spring Data JPA la integra mediante la interfaz `Pageable` y la clase `Page`.

```java
Page<Usuario> findByActivo(boolean activo, Pageable pageable);

List<Usuario> findByNombreContaining(String nombre, Sort sort);
```

El uso tipico desde el servicio:

```java
public Page<UsuarioDTO> listarUsuarios(int pagina, int tamano, String orden) {
    Pageable pageable = PageRequest.of(pagina, tamano, Sort.by(orden));
    Page<Usuario> usuarios = usuarioRepository.findByActivo(true, pageable);
    return usuarios.map(this::toDTO);
}
```

`Page` contiene no solo los resultados de la pagina actual, sino tambien metadatos: numero total de elementos, total de paginas, numero de pagina actual, si hay pagina siguiente o anterior. Esta informacion es esencial para construir controles de navegacion en el cliente.

```java
Page<Usuario> pagina = usuarioRepository.findAll(pageable);
int totalPaginas = pagina.getTotalPages();
long totalElementos = pagina.getTotalElements();
boolean haySiguiente = pagina.hasNext();
```

## Query Methods

Spring Data JPA deriva automaticamente la implementacion de un metodo a partir de su nombre, siguiendo una gramatica predecible. No es necesario escribir la query JPQL para los casos de uso mas comunes.

```java
// Busqueda por campo exacto
List<Usuario> findByNombre(String nombre);
Optional<Usuario> findByEmail(String email);

// Busqueda por multiplos campos con AND
List<Usuario> findByNombreAndApellido(String nombre, String apellido);

// Busqueda con condiciones
List<Usuario> findByEdadGreaterThan(int edad);
List<Usuario> findByEdadBetween(int min, int max);
List<Usuario> findByNombreContaining(String texto);
List<Usuario> findByFechaCreacionAfter(LocalDateTime fecha);
List<Usuario> findByActivoTrue();

// Agregacion
long countByActivo(boolean activo);
long countByCiudad(String ciudad);

// Existencia
boolean existsByEmail(String email);
boolean existsByNombreAndApellido(String nombre, String apellido);

// Eliminacion
void deleteByActivoFalse();
```

La gramatica de los Query Methods sigue el patron `[verbo][By][propiedad][Operador][And/Or][propiedad][Operador]`. Los verbos admitidos son `find...By`, `read...By`, `query...By`, `get...By`, `count...By`, `exists...By`, `delete...By`, `remove...By`. Los operadores incluyen `GreaterThan`, `LessThan`, `Between`, `Containing`, `Like`, `StartingWith`, `EndingWith`, `In`, `NotIn`, `IsNull`, `NotNull`, `True`, `False`, `OrderBy`, `IgnoreCase`.

```java
List<Usuario> findByNombreContainingIgnoreCase(String nombre);
List<Usuario> findByEdadGreaterThanOrderByNombreAsc(int edad);
List<Usuario> findByApellidoIn(List<String> apellidos);
```

> Los Query Methods son utiles para consultas simples, pero su legibilidad se degrada rapidamente al combinar mas de tres condiciones. Para consultas complejas es preferible usar `@Query` con JPQL explicito, y para condiciones puramente dinamicas (el usuario elige los filtros en tiempo de ejecucion), el patron Specification es la solucion adecuada.
