# Custom implementations

Aunque Spring Data JPA genera automaticamente la implementacion de los metodos declarados en las interfaces repository, existen escenarios donde es necesario proporcionar codigo personalizado. Esto incluye consultas excesivamente complejas que no pueden expresarse mediante `@Query`, logica que requiere acceso directo al `EntityManager`, o la integracion con APIs de terceros. Spring Data JPA ofrece un mecanismo de integracion elegante para combinar metodos generados con implementaciones propias.

## Interfaces custom

El primer paso consiste en definir una interfaz separada que declare los metodos personalizados. Esta interfaz es independiente de la jerarquia de Spring Data y sigue las convenciones estandar de Java:

```java
public interface UsuarioRepositoryCustom {
    List<Usuario> buscarUsuariosComplejo(String criterio, Integer limite);
    void actualizarUltimoAcceso(Long usuarioId);
}
```

## Implementacion con naming convention

Spring Data JPA busca automaticamente una clase de implementacion cuyo nombre siga la convencion `[InterfaceRepository]Impl`. La implementacion debe inyectar el `EntityManager` mediante `@PersistenceContext`:

```java
public class UsuarioRepositoryImpl implements UsuarioRepositoryCustom {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<Usuario> buscarUsuariosComplejo(String criterio, Integer limite) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Usuario> cq = cb.createQuery(Usuario.class);
        Root<Usuario> root = cq.from(Usuario.class);

        Predicate condicion = cb.like(
            cb.lower(root.get("nombre")),
            "%" + criterio.toLowerCase() + "%"
        );

        cq.where(condicion);
        cq.orderBy(cb.desc(root.get("fechaCreacion")));

        return entityManager.createQuery(cq)
            .setMaxResults(limite)
            .getResultList();
    }

    @Override
    public void actualizarUltimoAcceso(Long usuarioId) {
        entityManager.createQuery(
            "UPDATE Usuario u SET u.ultimoAcceso = CURRENT_TIMESTAMP WHERE u.id = :id")
            .setParameter("id", usuarioId)
            .executeUpdate();
    }
}
```

La implementacion debe estar en el mismo paquete que la interfaz del repository o ser escaneada por el mecanismo de componentes de Spring (anotada con `@Component` o detectada automaticamente).

## Integracion en el repository

El repository principal extiende tanto la interfaz generica de Spring Data como la interfaz custom:

```java
public interface UsuarioRepository extends
    JpaRepository<Usuario, Long>,
    UsuarioRepositoryCustom {
}
```

Spring Data JPA detecta que `UsuarioRepository` extiende `UsuarioRepositoryCustom`, localiza la implementacion `UsuarioRepositoryImpl` en el classpath, y genera un proxy que delega los metodos derivados al mecanismo estandar y los metodos personalizados a la implementacion proporcionada.

## Configuracion del sufijo

El sufijo por defecto para localizar la implementacion es `Impl`. Se puede cambiar globalmente en la configuracion:

```java
@EnableJpaRepositories(
    basePackages = "com.ejemplo.persistencia",
    repositoryImplementationPostfix = "CustomImpl"
)
@Configuration
public class PersistenciaConfig {
}
```

Con esta configuracion, Spring Data buscara `UsuarioRepositoryCustomImpl` en lugar de `UsuarioRepositoryImpl`.

> Si existen multiples implementaciones del mismo metodo personalizado (por ejemplo, en un fragmento y en la implementacion principal), la prioridad sigue reglas especificas. La implementacion principal siempre tiene preferencia sobre los fragmentos.

## Fragment repositories

Cuando se dispone de multiples aspectos de funcionalidad personalizada que no tienen relacion entre si, se pueden definir varias interfaces especializadas, cada una con su propia implementacion. Spring Data JPA las denomina fragment repositories:

```java
public interface UsuarioSearchFragment {
    List<Usuario> buscarPorNombreOEmail(String texto);
}

public interface UsuarioAuditFragment {
    void registrarAuditoria(Long usuarioId, String accion);
}
```

Cada fragmento tiene su propia implementacion siguiendo la convencion de nombres:

```java
public class UsuarioSearchFragmentImpl implements UsuarioSearchFragment {
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<Usuario> buscarPorNombreOEmail(String texto) {
        // implementacion
    }
}

public class UsuarioAuditFragmentImpl implements UsuarioAuditFragment {
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public void registrarAuditoria(Long usuarioId, String accion) {
        // implementacion
    }
}
```

El repository principal extiende todos los fragmentos junto con la interfaz de Spring Data:

```java
public interface UsuarioRepository extends
    JpaRepository<Usuario, Long>,
    UsuarioSearchFragment,
    UsuarioAuditFragment {
}
```

Spring Data JPA ensambla automaticamente los fragments en la implementacion final del proxy. Este enfoque sigue el principio de segregacion de interfaces (ISP) y facilita el mantenimiento al mantener cada aspecto de funcionalidad personalizada en su propio fragmento.

## Inyeccion del EntityManager

En las implementaciones personalizadas, el `EntityManager` se inyecta mediante `@PersistenceContext`. No se recomienda usar `@Autowired` para este proposito, ya que `@PersistenceContext` garantiza que el `EntityManager` sea seguro para el entorno transaccional y maneje correctamente la propagacion de contexto de persistencia.

```java
@PersistenceContext
private EntityManager entityManager;
```

## Implementacion con JdbcTemplate

Para operaciones que requieren acceso directo a JDBC o consultas SQL nativas, se puede inyectar `JdbcTemplate` en la implementacion personalizada:

```java
public class UsuarioRepositoryImpl implements UsuarioRepositoryCustom {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void actualizarMasivo(List<Long> ids) {
        String sql = "UPDATE usuarios SET activo = false WHERE id IN (:ids)";
        MapSqlParameterSource params = new MapSqlParameterSource("ids", ids);
        // ejecucion con NamedParameterJdbcTemplate
    }
}
```

Esto permite realizar operaciones masivas o consultas altamente optimizadas que no pasan por el mecanismo de JPA.
