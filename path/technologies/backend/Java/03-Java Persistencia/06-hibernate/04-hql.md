# HQL

HQL (Hibernate Query Language) es el lenguaje de consulta nativo de Hibernate. Su sintaxis es semanticamente similar a JPQL (Java Persistence Query Language), pero Hibernate lo extiende con funcionalidades adicionales como soporte para funciones propias de base de datos, `ScrollableResults` para recorrido eficiente de grandes conjuntos de resultados y la capacidad de ejecutar consultas de mutacion (UPDATE y DELETE directamente sobre entidades).

A diferencia de SQL, que trabaja sobre tablas y columnas, HQL trabaja sobre entidades y sus propiedades. La consulta se expresa en terminos del modelo de objetos y Hibernate se encarga de traducirla al SQL optimo para el dialecto configurado.

```java
String hql = "FROM Usuario u WHERE u.email = :email";
List<Usuario> resultados = session.createQuery(hql, Usuario.class)
    .setParameter("email", "usuario@ejemplo.com")
    .list();
```

## Parametros nombrados y posicionales

HQL soporta dos formas de vinculacion de parametros. Los parametros nombrados usan `:nombre` y son la opcion recomendada por su claridad y porque el orden de los parametros no importa.

```java
Query<Pedido> query = session.createQuery(
    "FROM Pedido p WHERE p.cliente.id = :clienteId AND p.total > :totalMinimo",
    Pedido.class);
query.setParameter("clienteId", 42L);
query.setParameter("totalMinimo", BigDecimal.valueOf(100));
List<Pedido> pedidos = query.list();
```

Los parametros posicionales utilizan `?` con un indice numerico. Aunque funcionan, son mas fragiles porque el indice debe coincidir exactamente con la posicion en la consulta.

```java
Query<Pedido> query = session.createQuery(
    "FROM Pedido p WHERE p.cliente.id = ?1 AND p.total > ?2",
    Pedido.class);
query.setParameter(1, 42L);
query.setParameter(2, BigDecimal.valueOf(100));
```

> El uso de `?` sin indice numerico (parametros posicionales clasicos) esta deprecado desde Hibernate 5. Se recomienda usar siempre parametros nombrados o parametros con indice explicito (`?1`, `?2`, etc.).

## Resultados unicos y paginacion

Para consultas que devuelven un unico resultado, `uniqueResult()` lanza una excepcion si hay mas de un resultado.

```java
Usuario usuario = session.createQuery(
    "FROM Usuario u WHERE u.email = :email", Usuario.class)
    .setParameter("email", "admin@ejemplo.com")
    .uniqueResult();
```

La paginacion se maneja con `setFirstResult()` y `setMaxResults()`, que Hibernate traduce a clausulas LIMIT/OFFSET o ROW_NUMBER segun el dialecto.

```java
List<Producto> productos = session.createQuery(
    "FROM Producto p ORDER BY p.nombre", Producto.class)
    .setFirstResult(20)
    .setMaxResults(10)
    .list();
```

## ScrollableResults

Cuando una consulta puede devolver millones de registros, cargar todos en memoria con `list()` es inviable. `ScrollableResults` permite recorrer el resultado de forma eficiente utilizando un cursor en el driver JDBC, manteniendo solo una parte de los datos en memoria.

```java
try (ScrollableResults<Proceso> scroll = session.createQuery(
        "FROM Proceso p WHERE p.estado = 'PENDIENTE'", Proceso.class)
        .scroll(ScrollMode.FORWARD_ONLY)) {
    while (scroll.next()) {
        Proceso p = scroll.get();
        p.ejecutar();
        session.update(p);
        if (++contador % 100 == 0) {
            session.flush();
            session.clear();
        }
    }
}
```

> `ScrollMode.FORWARD_ONLY` combinado con `flush()` y `clear()` periodicos es la tecnica estandar para procesamiento por lotes (batch processing) en Hibernate, evitando el desbordamiento del first-level cache.

## Consultas de mutacion (UPDATE y DELETE)

HQL permite ejecutar sentencias UPDATE y DELETE directamente, sin necesidad de cargar las entidades en memoria. Esto es mucho mas eficiente para operaciones masivas.

```java
int actualizados = session.createMutationQuery(
    "UPDATE Producto p SET p.precio = p.precio * 1.10 WHERE p.categoria = :cat")
    .setParameter("cat", "ELECTRONICA")
    .executeUpdate();

int eliminados = session.createMutationQuery(
    "DELETE FROM Producto p WHERE p.stock = 0")
    .executeUpdate();
```

> Las consultas de mutacion no actualizan el first-level cache ni el second-level cache. Despues de ejecutar una mutacion, las entidades afectadas que ya estuvieran cargadas en la sesion quedaran en un estado inconsistente. Es responsabilidad del desarrollador hacer `clear()` o `refresh()` sobre las entidades afectadas.

## Funciones y expresiones

HQL expone un conjunto amplio de funciones que pueden variar segun el dialecto configurado. Entre las mas utilizadas estan `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()`, `CONCAT()`, `SUBSTRING()`, `UPPER()`, `LOWER()`, `TRIM()`, `LENGTH()`, `COALESCE()`, y `CASE WHEN`.

```java
List<Object[]> resultados = session.createQuery(
    "SELECT p.categoria, COUNT(p), AVG(p.precio) " +
    "FROM Producto p " +
    "GROUP BY p.categoria " +
    "HAVING COUNT(p) > 10", Object[].class)
    .list();
```

Las funciones de agregacion y las expresiones `GROUP BY`/`HAVING` se comportan igual que en SQL, pero operando sobre propiedades de entidades en lugar de columnas.
