# @Query annotation

La anotacion `@Query` permite definir consultas personalizadas de forma explicita cuando la query derivation no es suficiente o cuando se necesita control total sobre la sentencia JPQL o SQL que se ejecuta. Mientras que la derivacion por nombre es adecuada para consultas simples, `@Query` brinda la flexibilidad necesaria para consultas complejas, optimizaciones especificas o la utilizacion de funciones nativas de la base de datos.

## JPQL con @Query

El uso mas comun de `@Query` es la definicion de consultas JPQL directamente sobre el metodo del repository. La consulta se escribe como una cadena en el atributo `value` de la anotacion:

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {

    @Query("SELECT u FROM Usuario u WHERE u.email = ?1")
    Optional<Usuario> buscarPorEmail(String email);

    @Query("SELECT u FROM Usuario u WHERE u.activo = true AND u.edad >= ?1")
    List<Usuario> buscarUsuariosActivosMayoresDe(Integer edad);
}
```

Los parametros se referencian por posicion usando `?1`, `?2`, etc., donde el numero indica la posicion del parametro en la firma del metodo. Este enfoque posicional es simple pero puede volverse propenso a errores cuando hay muchos parametros.

## Parametros nombrados con @Param

Una alternativa mas legible y mantenible es el uso de parametros nombrados. Se utiliza `:nombre` en la consulta JPQL y se anota cada parametro del metodo con `@Param("nombre")`:

```java
@Query("SELECT u FROM Usuario u WHERE u.email = :email AND u.activo = :activo")
Optional<Usuario> buscarPorEmailYActivo(
    @Param("email") String email,
    @Param("activo") Boolean activo
);
```

Los parametros nombrados mejoran la legibilidad de la consulta y eliminan la dependencia del orden de los parametros en la firma del metodo. Si se intercambian los parametros en el metodo, la consulta sigue funcionando correctamente.

## Consultas nativas

El atributo `nativeQuery = true` indica que la consulta esta escrita en SQL nativo de la base de datos subyacente, en lugar de JPQL:

```java
@Query(value = "SELECT * FROM usuarios WHERE email = ?1 AND activo = ?2",
       nativeQuery = true)
Optional<Usuario> buscarPorEmailNativo(String email, Boolean activo);
```

Las consultas nativas son necesarias cuando se utilizan funciones especificas del motor de base de datos (como `JSON_EXTRACT` en MySQL, `ARRAY_AGG` en PostgreSQL, funciones de ventana, consultas recursivas con CTE) o cuando se necesita optimizar una consulta que el generador JPQL no produce de forma eficiente.

> Las consultas nativas devuelven resultados que no pasan por el cache de primer nivel de Hibernate a menos que se especifique explícitamente. Ademas, el mapeo de resultados puede requerir atencion adicional si las columnas devueltas no coinciden exactamente con las propiedades de la entidad.

## @Modifying para UPDATE y DELETE

Las operaciones que modifican la base de datos (UPDATE, DELETE) deben anotarse adicionalmente con `@Modifying`. Sin esta anotacion, Spring Data JPA lanza una excepcion porque espera que toda consulta con `@Query` sea de tipo SELECT.

```java
@Modifying
@Query("UPDATE Usuario u SET u.activo = false WHERE u.id = :id")
int desactivarUsuario(@Param("id") Long id);

@Modifying
@Query("DELETE FROM Usuario u WHERE u.email = :email")
int eliminarPorEmail(@Param("email") String email);
```

El valor de retorno de las operaciones `@Modifying` es `int` o `void`. El entero indica el numero de filas afectadas por la operacion.

`@Modifying` tiene un atributo `flushAutomatically` que, cuando se establece a `true`, fuerza el flush del `EntityManager` antes de ejecutar la consulta. Esto evita inconsistencias si hay cambios pendientes en el contexto de persistencia:

```java
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("UPDATE Usuario u SET u.activo = false WHERE u.id = :id")
int desactivarUsuario(@Param("id") Long id);
```

`clearAutomatically = true` limpia el cache de primer nivel despues de la ejecucion, asegurando que las entidades gestionadas no queden en un estado obsoleto.

> Sin `clearAutomatically = true`, las entidades cargadas previamente en el contexto de persistencia pueden mantener valores antiguos incluso despues de que la base de datos haya sido actualizada. Esto provoca errores de lectura sucia a nivel de aplicacion.

## Paginacion con @Query

Los metodos anotados con `@Query` pueden aceptar parametros `Pageable` y `Sort` para integrar paginacion y ordenamiento:

```java
@Query("SELECT u FROM Usuario u WHERE u.activo = true")
Page<Usuario> buscarActivos(Pageable pageable);
```

Spring Data JPA transforma automaticamente la consulta base para anadir las clausulas `LIMIT`, `OFFSET` y `ORDER BY` correspondientes, y ejecuta una consulta `SELECT COUNT(*)` adicional para calcular el total de elementos.

## Named queries

Como alternativa a `@Query` en el repository, las consultas pueden definirse directamente en la entidad mediante `@NamedQuery`:

```java
@Entity
@NamedQuery(name = "Usuario.buscarPorEmail",
            query = "SELECT u FROM Usuario u WHERE u.email = ?1")
public class Usuario {
    // ...
}
```

Y en el repository, el metodo se declara sin anotacion, confiando en que Spring Data JPA resuelva automaticamente la named query por convencion:

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    Usuario buscarPorEmail(String email);
}
```

El nombre del metodo debe coincidir con el nombre definido en `@NamedQuery` siguiendo el patron `NombreEntidad.nombreMetodo`.

## @Query con proyeccion

`@Query` admite la especificacion de proyecciones personalizadas mediante expresiones JPQL en el SELECT:

```java
@Query("SELECT u.nombre, u.email FROM Usuario u WHERE u.activo = true")
List<Object[]> buscarNombresYEmails();

@Query("SELECT new com.ejemplo.dto.UsuarioResumen(u.nombre, u.email) FROM Usuario u")
List<UsuarioResumen> buscarResumen();
```

La segunda forma utiliza el constructor de un DTO en la propia consulta JPQL, devolviendo directamente objetos del tipo deseado sin necesidad de mapeo posterior.
