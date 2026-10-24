# SQL nativo en Hibernate

Aunque HQL y Criteria API cubren la mayoria de las necesidades de consulta, existen escenarios donde se requiere SQL nativo: consultas altamente optimizadas para el motor concreto, funciones especificas de la base de datos (como `FULLTEXT` en MySQL o `JSONB` en PostgreSQL), procedimientos almacenados o migracion de codigo JDBC existente. Hibernate proporciona `createNativeQuery()` para este proposito.

## Consultas basicas con SQL nativo

El metodo `createNativeQuery()` acepta una cadena SQL y devuelve una `Query` que puede ejecutarse como cualquier consulta HQL.

```java
List<Object[]> resultados = session.createNativeQuery(
    "SELECT id, nombre, email FROM usuarios WHERE activo = true")
    .list();

for (Object[] fila : resultados) {
    Long id = (Long) fila[0];
    String nombre = (String) fila[1];
    String email = (String) fila[2];
}
```

Cuando la consulta devuelve varias columnas, el resultado es una lista de arrays de `Object`. El orden de las columnas debe coincidir con el indice del array.

## Mapeo a entidades con addEntity()

Para que Hibernate mapee automaticamente cada fila del resultado SQL a una entidad JPA, se usa `addEntity()`:

```java
List<Usuario> usuarios = session.createNativeQuery(
    "SELECT * FROM usuarios WHERE email LIKE :dominio")
    .addEntity(Usuario.class)
    .setParameter("dominio", "%@empresa.com")
    .list();
```

> `addEntity()` requiere que la consulta SQL devuelva todas las columnas que la entidad necesita. Si falta alguna columna obligatoria (nullable = false), Hibernate lanzara una excepcion.

Para consultas que involucran varias entidades, se pueden mapear multiples entidades:

```java
List<Object[]> resultados = session.createNativeQuery(
    "SELECT u.*, p.* FROM usuarios u JOIN pedidos p ON u.id = p.usuario_id " +
    "WHERE p.total > 1000")
    .addEntity("usuario", Usuario.class)
    .addEntity("pedido", Pedido.class)
    .list();

for (Object[] fila : resultados) {
    Usuario u = (Usuario) fila[0];
    Pedido p = (Pedido) fila[1];
}
```

## Mapeo escalar con addScalar()

`addScalar()` permite controlar como se mapea cada columna individual, especificando el tipo de Hibernate para evitar ambiguedades. Es util cuando el SQL nativo devuelve funciones de agregacion, expresiones o tipos que Hibernate no puede inferir automaticamente.

```java
List<Object[]> resultados = session.createNativeQuery(
    "SELECT COUNT(*) as total, AVG(precio) as promedio FROM productos")
    .addScalar("total", StandardBasicTypes.LONG)
    .addScalar("promedio", StandardBasicTypes.BIG_DECIMAL)
    .list();
```

> Sin `addScalar()`, Hibernate intenta inferir el tipo de cada columna del resultado, lo que puede fallar con funciones de base de datos que devuelven tipos no estandar. `addScalar()` elimina la ambiguedad y ademas mejora el rendimiento porque Hibernate no necesita analizar los metadatos del ResultSet.

## Parameter binding

Los parametros en SQL nativo se vinculan igual que en HQL, usando `:` para parametros nombrados o `?` para posicionales:

```java
// Parametros nombrados
List<Producto> productos = session.createNativeQuery(
    "SELECT * FROM productos WHERE precio BETWEEN :min AND :max " +
    "AND categoria = :cat", Producto.class)
    .setParameter("min", BigDecimal.valueOf(10))
    .setParameter("max", BigDecimal.valueOf(100))
    .setParameter("cat", "ELECTRONICA")
    .list();

// Parametros posicionales
List<Producto> productos = session.createNativeQuery(
    "SELECT * FROM productos WHERE precio BETWEEN ?1 AND ?2", Producto.class)
    .setParameter(1, BigDecimal.valueOf(10))
    .setParameter(2, BigDecimal.valueOf(100))
    .list();
```

> Al igual que en HQL, los parametros posicionales con indice explicito (`?1`, `?2`) son preferibles a los parametros posicionales clasicos (`?`) que estan deprecados.

## Stored procedures

Hibernate soporta la ejecucion de procedimientos almacenados mediante `createStoredProcedureQuery()`:

```java
StoredProcedureQuery query = session.createStoredProcedureQuery("sp_actualizar_stock");
query.registerStoredProcedureParameter("producto_id", Long.class, ParameterMode.IN);
query.registerStoredProcedureParameter("cantidad", Integer.class, ParameterMode.IN);
query.registerStoredProcedureParameter("resultado", Boolean.class, ParameterMode.OUT);

query.setParameter("producto_id", 123L);
query.setParameter("cantidad", 10);

boolean exito = query.execute();
Boolean resultado = (Boolean) query.getOutputParameterValue("resultado");
```

Tambien se puede usar SQL nativo con `CALL`:

```java
session.createNativeQuery("CALL sp_actualizar_stock(:pid, :cant)")
    .setParameter("pid", 123L)
    .setParameter("cant", 10)
    .executeUpdate();
```

> Para procedimientos que devuelven conjuntos de resultados (ResultSet), se usa `execute()` y luego se itera sobre los resultados con `getResultList()`. El soporte de cursores y multiples ResultSets depende del driver JDBC y del motor de base de datos.

## Result set mapping

Para escenarios complejos donde el SQL nativo no se alinea directamente con las entidades, se puede definir un `@SqlResultSetMapping` en la entidad o en una clase separada:

```java
@SqlResultSetMapping(
    name = "ResumenProductoMapping",
    entities = @EntityResult(entityClass = Producto.class),
    columns = @ColumnResult(name = "total_vendido", type = Long.class)
)
@Entity
public class Producto {
    // ...
}
```

Y usarlo en la consulta nativa:

```java
List<Object[]> resultados = session.createNativeQuery(
    "SELECT p.*, SUM(lp.cantidad) as total_vendido " +
    "FROM productos p " +
    "JOIN lineas_pedido lp ON p.id = lp.producto_id " +
    "GROUP BY p.id",
    "ResumenProductoMapping")
    .list();
```

Cada fila del resultado contiene la entidad `Producto` completa en la primera posicion y el valor agregado `total_vendido` en la segunda.

## Ejecutar sentencias de mutacion

`executeUpdate()` en SQL nativo funciona igual que en HQL:

```java
int actualizados = session.createNativeQuery(
    "UPDATE productos SET precio = precio * 1.10 " +
    "WHERE categoria = :categoria")
    .setParameter("categoria", "ELECTRONICA")
    .executeUpdate();
```

> Al igual que con las mutaciones HQL, las entidades afectadas que ya esten cargadas en la sesion no se actualizan automaticamente. Es necesario hacer `session.refresh(entity)` o `session.clear()` para mantener la consistencia.
