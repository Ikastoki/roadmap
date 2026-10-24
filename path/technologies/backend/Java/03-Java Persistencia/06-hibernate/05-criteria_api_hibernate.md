# Criteria API de Hibernate

La Criteria API es una alternativa programatica a HQL/JPQL que permite construir consultas de forma dinamica y tipada usando objetos Java en lugar de cadenas de texto. Esto es especialmente util cuando la estructura de la consulta no se conoce en tiempo de compilacion, como ocurre en filtros de busqueda avanzada o paneles de administracion con multiples criterios opcionales.

Hibernate implementa la Criteria API de JPA (`jakarta.persistence.criteria`) y la extiende con funcionalidades adicionales. Las clases centrales son `CriteriaBuilder`, `CriteriaQuery`, `Root`, `Join`, `Predicate` y `Expression`.

## CriteriaBuilder y CriteriaQuery

El punto de entrada es `CriteriaBuilder`, que se obtiene del `EntityManager` o de la `Session`. A partir de el se crea un objeto `CriteriaQuery<T>` que define la estructura de la consulta.

```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<Usuario> query = cb.createQuery(Usuario.class);
Root<Usuario> root = query.from(Usuario.class);
query.select(root);

List<Usuario> usuarios = session.createQuery(query).list();
```

El ejemplo anterior equivale a `FROM Usuario`. La consulta es completamente segura en cuanto a tipos: el compilador verifica que las propiedades referenciadas existan y que los tipos de los parametros coincidan.

## Predicados y Expresiones

Los predicados son las condiciones WHERE de la consulta. Se construyen con metodos de `CriteriaBuilder` como `equal()`, `notEqual()`, `like()`, `greaterThan()`, `lessThanOrEqualTo()`, `isNull()`, `isNotNull()`, `between()`, `in()`, etc.

```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<Usuario> query = cb.createQuery(Usuario.class);
Root<Usuario> root = query.from(Usuario.class);

Predicate activo = cb.equal(root.get("activo"), true);
Predicate nombreInicia = cb.like(root.get("nombre"), "A%");
Predicate fechaReciente = cb.greaterThan(
    root.get("fechaRegistro"), LocalDate.now().minusMonths(1));

query.select(root)
     .where(cb.and(activo, nombreInicia, fechaReciente))
     .orderBy(cb.asc(root.get("nombre")));

List<Usuario> resultado = session.createQuery(query).list();
```

Cuando los predicados son opcionales, se pueden construir dinamicamente:

```java
List<Predicate> predicados = new ArrayList<>();
predicados.add(cb.equal(root.get("activo"), true));

if (nombreBusqueda != null && !nombreBusqueda.isEmpty()) {
    predicados.add(cb.like(root.get("nombre"), "%" + nombreBusqueda + "%"));
}
if (categoriaId != null) {
    predicados.add(cb.equal(root.get("categoria").get("id"), categoriaId));
}

query.where(cb.and(predicados.toArray(new Predicate[0])));
```

> El enfoque de construir una lista de predicados y convertirla a array con `toArray(new Predicate[0])` es el patron mas comun para filtros dinamicos. `cb.and()` y `cb.or()` pueden combinarse para producir condiciones compuestas.

## Joins

Las relaciones entre entidades se navegan mediante `join()`, que acepta el nombre de la propiedad que define la relacion. Existen variantes para cada tipo de join: `join()` (INNER JOIN), `leftJoin()` (LEFT JOIN), `rightJoin()` (RIGHT JOIN) y `fetch()` (JOIN FETCH para evitar N+1).

```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<Pedido> query = cb.createQuery(Pedido.class);
Root<Pedido> root = query.from(Pedido.class);
Join<Pedido, Cliente> cliente = root.join("cliente");
Join<Pedido, LineaPedido> lineas = root.join("lineas");

query.select(root)
     .where(cb.and(
         cb.equal(cliente.get("email"), "cliente@ejemplo.com"),
         cb.greaterThan(lineas.get("cantidad"), 1)
     ));
```

Para evitar el problema N+1 en consultas que luego van a navegar las colecciones, se usa `fetch()`:

```java
Root<Pedido> root = query.from(Pedido.class);
root.fetch("lineas", JoinType.LEFT);
query.select(root).distinct(true);
```

> `distinct(true)` es necesario cuando se hace fetch de colecciones porque de lo contrario Hibernate devolvera filas duplicadas, una por cada elemento de la coleccion.

## Subqueries

Las subconsultas se construyen con `CriteriaBuilder.createSubquery()` y se integran en los predicados mediante operadores como `in()`, `exists()`, `all()`, `any()`.

```java
CriteriaBuilder cb = session.getCriteriaBuilder();

CriteriaQuery<Producto> query = cb.createQuery(Producto.class);
Root<Producto> root = query.from(Producto.class);

Subquery<Long> subquery = query.subquery(Long.class);
Root<Pedido> subRoot = subquery.from(Pedido.class);
Join<Pedido, LineaPedido> subLineas = subRoot.join("lineas");
subquery.select(subLineas.get("producto").get("id"))
        .where(cb.greaterThan(subLineas.get("cantidad"), 100));

query.select(root)
     .where(cb.in(root.get("id")).value(subquery));
```

## Group by y Having

La Criteria API soporta agrupaciones y filtros sobre grupos:

```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<Tuple> query = cb.createQuery(Tuple.class);
Root<Producto> root = query.from(Producto.class);

query.multiselect(
    root.get("categoria"),
    cb.count(root),
    cb.avg(root.get("precio"))
).groupBy(root.get("categoria"))
 .having(cb.greaterThan(cb.count(root), 5L));

List<Tuple> resultados = session.createQuery(query).list();
for (Tuple t : resultados) {
    System.out.println(t.get(0) + " - " + t.get(1) + " - " + t.get(2));
}
```

## Ordenacion

La ordenacion se define con `orderBy()` accepts `cb.asc()` y `cb.desc()`:

```java
query.select(root)
     .where(cb.equal(root.get("activo"), true))
     .orderBy(
         cb.asc(root.get("categoria")),
         cb.desc(root.get("fechaCreacion"))
     );
```

> La Criteria API es mas verbosa que HQL para consultas simples, pero indispensable cuando la consulta se construye dinamicamente en base a parametros opcionales, como en sistemas de busqueda con multiples filtros.
