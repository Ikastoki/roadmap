# DAO vs Repository

DAO (Data Access Object) y Repository son dos patrones que resuelven el mismo problema fundamental —separar la logica de acceso a datos del resto de la aplicacion— pero con filosofias y niveles de abstraccion distintos. La confusion entre ambos es frecuente porque en la practica muchas implementaciones llamadas "Repository" son en realidad DAOs, y viceversa. Comprender las diferencias requiere examinar no solo la estructura del codigo, sino el origen y la intencion de cada patron.

## DAO (Data Access Object)

El patron DAO surge en los anos 2000 con J2EE como una solucion para aislar el codigo JDBC del resto de la aplicacion. Su objetivo es encapsular el acceso a una tabla o vista de la base de datos, exponiendo un conjunto de operaciones CRUD que ocultan los detalles de la conexion, la construccion de sentencias SQL y el mapeo de resultados.

```java
public interface UsuarioDAO {
    Usuario findById(Long id);
    List<Usuario> findAll();
    Usuario save(Usuario usuario);
    void update(Usuario usuario);
    void deleteById(Long id);
}

public class UsuarioDAOImpl implements UsuarioDAO {
    private final DataSource dataSource;

    public UsuarioDAOImpl(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Usuario findById(Long id) {
        String sql = "SELECT * FROM usuarios WHERE id = ?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setLong(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return mapRow(rs);
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
        return null;
    }

    private Usuario mapRow(ResultSet rs) throws SQLException {
        Usuario u = new Usuario();
        u.setId(rs.getLong("id"));
        u.setNombre(rs.getString("nombre"));
        u.setEmail(rs.getString("email"));
        return u;
    }
}
```

El DAO trabaja en el nivel de la tabla: cada metodo encapsula una sentencia SQL y mapea el resultado a objetos. No hay nocion de coleccion ni de identidad de objeto; la entidad es simplemente el destinatario de los datos extraidos de la fila.

## Repository

El patron Repository, popularizado por Eric Evans en Domain-Driven Design, abstrae una coleccion de objetos del dominio, no el acceso a una tabla. La interfaz del Repository utiliza el lenguaje del dominio y trata las entidades como ciudadanos de primera clase con identidad y ciclo de vida.

```java
// Interfaz en la capa de dominio
public interface UsuarioRepository {
    Optional<Usuario> findByEmail(String email);
    List<Usuario> findActiveUsers();
    Usuario save(Usuario usuario);
    void delete(Usuario usuario);
}

// Implementacion en la capa de persistencia
public class UsuarioRepositoryImpl implements UsuarioRepository {
    private final EntityManager em;

    public UsuarioRepositoryImpl(EntityManager em) {
        this.em = em;
    }

    @Override
    public Optional<Usuario> findByEmail(String email) {
        return em.createQuery("SELECT u FROM Usuario u WHERE u.email = :email", Usuario.class)
                .setParameter("email", email)
                .getResultStream()
                .findFirst();
    }

    @Override
    public List<Usuario> findActiveUsers() {
        return em.createQuery("SELECT u FROM Usuario u WHERE u.activo = true", Usuario.class)
                .getResultList();
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
    public void delete(Usuario usuario) {
        em.remove(em.contains(usuario) ? usuario : em.merge(usuario));
    }
}
```

La interfaz reside en la capa de dominio, junto a las entidades de negocio. Esto permite que el servicio de negocio dependa de una abstraccion pura, sin ninguna referencia a JPA, JDBC o cualquier tecnologia de persistencia. El Repository devuelve objetos del dominio, no registros de base de datos.

## Diferencias clave

| Aspecto | DAO | Repository |
|---|---|---|
| Abstraccion | Tabla o vista de BD | Coleccion de objetos del dominio |
| Lenguaje | SQL (tablas, columnas, joins) | Lenguaje del dominio (entidades, relaciones de negocio) |
| Orientacion | Datos | Objetos |
| Interfaz | En capa de persistencia | En capa de dominio |
| Identidad | Gestionada por la aplicacion | Gestionada por el Repository |
| Reutilizacion | Baja (cada tabla tiene su DAO) | Alta (operaciones semanticas del dominio) |
| Testing | Requiere base de datos o mock pesado | Facil de mockear (interfaz en dominio) |

La diferencia fundamental no es tecnica sino arquitectonica. Un DAO es un detalle de implementacion: su lugar natural esta en la capa de infraestructura, y cambiarlo implica modificar solo la persistencia. Un Repository es una abstraccion del dominio: su interfaz es parte del lenguaje ubicuo, y su implementacion es intercambiable sin que el dominio se entere.

```java
// DAO: opera en terminos de SQL
public interface UsuarioDAO {
    List<Usuario> findBySQL(String whereClause, Object... params);
    int updateBySQL(String sql, Object... params);
}

// Repository: opera en terminos del dominio
public interface UsuarioRepository {
    List<Usuario> findPremiumUsers();
    Optional<Usuario> findByIdWithOrders(Long id);
    void deactivateInactiveUsers(int daysSinceLastLogin);
}
```

## Cuando usar cada uno

El Repository es el patron recomendado en aplicaciones que siguen Domain-Driven Design, arquitectura hexagonal, clean architecture, o cualquier enfoque donde el dominio sea el nucleo del sistema. Es la opcion natural cuando se usa JPA y Spring Data, ya que `JpaRepository` es precisamente una implementacion del patron Repository.

El DAO sigue siendo util en escenarios donde no existe un modelo de dominio rico: aplicaciones CRUD simples, reportes, procesos batch que operan directamente sobre tablas, o cuando se trabaja con SQL nativo complejo que no se beneficia del mapeo objeto-relacional. En estos casos, un DAO que encapsula las consultas SQL es mas directo y transparente que forzar un Repository que oculte detras de si un `@Query` nativo igualmente acoplado a la base de datos.

En la practica, muchas aplicaciones combinan ambos: Repository para las operaciones del dominio (gestion de usuarios, pedidos, productos) y DAO para consultas de reporte, analitica o integracion con sistemas legacy donde la abstraccion objeto-relacional no aporta valor.

> Usar Spring Data `JpaRepository` y llamarlo Repository es correcto, pero solo si la interfaz expresa operaciones del dominio. Un `JpaRepository<Usuario, Long>` que solo expone `findAll`, `findById` y `save` es, arquitectonicamente, un DAO disfrazado de Repository. La diferencia no esta en la tecnologia sino en el nivel de abstraccion de los metodos que se exponen.
