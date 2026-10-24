# Repository interfaces

Spring Data JPA organiza su jerarquia de interfaces alrededor de tres niveles principales: `CrudRepository`, `PagingAndSortingRepository` y `JpaRepository`. Cada nivel anade funcionalidad sobre el anterior, permitiendo al desarrollador elegir el grado de abstraccion que mejor se adapte a sus necesidades. Comprender las capacidades de cada una de estas interfaces es fundamental para disenar una capa de persistencia eficiente.

## CrudRepository

`CrudRepository<T, ID>` es la interfaz base del sistema. Proporciona las operaciones fundamentales de creacion, lectura, actualizacion y eliminacion para una entidad de tipo `T` con una clave primaria de tipo `ID`. Es la interfaz mas generica y ligera, ideal cuando solo se necesitan operaciones basicas.

```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);
    <S extends T> Iterable<S> saveAll(Iterable<S> entities);
    Optional<T> findById(ID id);
    boolean existsById(ID id);
    Iterable<T> findAll();
    Iterable<T> findAllById(Iterable<ID> ids);
    long count();
    void deleteById(ID id);
    void delete(T entity);
    void deleteAll(Iterable<? extends T> entities);
    void deleteAll();
}
```

El metodo `save()` funciona como un merge inteligente: si la entidad tiene un ID nulo o cero (segun el tipo de generacion de ID), ejecuta un `persist`; si el ID ya existe en la base de datos, ejecuta un `merge`. Este comportamiento convierte a `save()` en un metodo idempotente para entidades gestionadas.

`findById()` devuelve un `Optional<T>`, lo que obliga al cliente a manejar explicitamente el caso de que la entidad no exista. Esto elimina la necesidad de comprobaciones `null` y fomenta un codigo mas seguro.

```java
Usuario usuario = usuarioRepository.findById(1L)
    .orElseThrow(() -> new UsuarioNoEncontradoException("Usuario no encontrado"));
```

`deleteById()` lanza una `IllegalArgumentException` si el ID es nulo, pero no falla si el ID no existe en la base de datos. Simplemente no elimina nada.

> CrudRepository no depende de ninguna funcionalidad especifica de JPA. Esto significa que las implementaciones de Spring Data para otros almacenes (MongoDB, Redis, Cassandra) tambien extienden esta misma interfaz, ofreciendo una API uniforme independientemente del motor de persistencia subyacente.

## PagingAndSortingRepository

`PagingAndSortingRepository<T, ID>` extiende `CrudRepository` y anade soporte para paginacion y ordenamiento. Es la interfaz adecuada cuando las consultas pueden devolver un numero potencialmente grande de resultados y es necesario limitar el tamaño de las respuestas.

```java
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}
```

`findAll(Sort sort)` permite ordenar los resultados segun uno o varios campos, en direccion ascendente o descendente. El parametro `Sort` admite una sintaxis fluida:

```java
Sort orden = Sort.by("apellido").ascending()
                 .and(Sort.by("nombre").descending());
List<Usuario> usuarios = usuarioRepository.findAll(orden);
```

`findAll(Pageable pageable)` combina paginacion y ordenamiento en un solo parametro. `Pageable` es una interfaz cuyas implementaciones mas comunes se crean mediante `PageRequest.of()`:

```java
Pageable paginacion = PageRequest.of(0, 10, Sort.by("apellido").ascending());
Page<Usuario> pagina = usuarioRepository.findAll(paginacion);
```

La diferencia fundamental entre `Page<T>` y `Iterable<T>` es que `Page<T>` contiene metadatos adicionales como el numero total de elementos, el numero total de paginas y si existe una pagina siguiente o anterior. Sin embargo, obtener el recuento total requiere una consulta `SELECT COUNT(*)` adicional, lo que puede tener implicaciones de rendimiento en conjuntos de datos muy grandes.

## JpaRepository

`JpaRepository<T, ID>` es la interfaz mas completa. Extiende `PagingAndSortingRepository` y, a traves de herencia indirecta, `CrudRepository`. Esta disenada especificamente para JPA y expone metodos que requieren acceso al `EntityManager` subyacente.

```java
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>,
                                              QueryByExampleExecutor<T> {
    List<T> findAll();
    List<T> findAll(Sort sort);
    List<T> findAllById(Iterable<ID> ids);
    <S extends T> List<S> saveAll(Iterable<S> entities);
    void flush();
    <S extends T> S saveAndFlush(S entity);
    <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);
    void deleteAllInBatch(Iterable<T> entities);
    void deleteAllByIdInBatch(Iterable<ID> ids);
    void deleteAllInBatch();
    T getOne(ID id);
    T getReferenceById(ID id);
    T getById(ID id);
    <S extends T> Optional<S> findOne(Example<S> example);
    <S extends T> List<S> findAll(Example<S> example);
}
```

`flush()` sincroniza el estado del contexto de persistencia con la base de datos sin cerrar la transaccion. Fuerza la ejecucion de todas las sentencias SQL pendientes en el `EntityManager`. Es util cuando se necesita asegurar que ciertos cambios se reflejen en la base de datos antes de continuar con la siguiente operacion.

`saveAndFlush()` combina `save()` y `flush()` en un solo paso: persiste la entidad y fuerza la sincronizacion inmediata con la base de datos. El valor de retorno es la entidad gestionada actualizada, lo que permite obtener el ID generado si se utiliza generacion automatica.

```java
Usuario usuario = new Usuario("juan@email.com");
Usuario usuarioPersistido = usuarioRepository.saveAndFlush(usuario);
Long idGenerado = usuarioPersistido.getId();
```

`getReferenceById()` (antes `getOne()`) devuelve una referencia proxy a la entidad sin cargar realmente los datos de la base de datos. Solo se lanza una consulta cuando se accede a una propiedad distinta del ID. Esto es util para establecer relaciones entre entidades sin necesidad de cargar la entidad completa.

```java
Pedido pedido = new Pedido();
Usuario referencia = usuarioRepository.getReferenceById(1L);
pedido.setUsuario(referencia);
```

En este ejemplo, no se ejecuta ninguna consulta `SELECT` contra la tabla de usuarios. Hibernate crea un proxy que solo se materializara si se accede a alguna propiedad del usuario.

`deleteAllInBatch()` elimina todas las entidades en una sola sentencia SQL, a diferencia de `deleteAll()` de `CrudRepository` que las elimina una por una. La version "in batch" no realiza la carga previa de las entidades en memoria.

> `getReferenceById()` lanza una `EntityNotFoundException` (de Hibernate) si la entidad no existe, pero solo en el momento en que se accede a una propiedad distinta del ID. Este comportamiento diferido puede provocar errores sorprendentes si no se maneja adecuadamente.
