# Consultas con JPA

JPA ofrece multiples mecanismos para consultar la base de datos, desde un lenguaje de consultas propio hasta la posibilidad de ejecutar SQL nativo. Cada enfoque tiene su nicho: JPQL para consultas orientadas a objetos, Criteria API para consultas dinamicas tipadas, y consultas nativas cuando se necesita control total sobre el SQL generado. La eleccion determina no solo la legibilidad del codigo, sino tambien la portabilidad entre bases de datos y la capacidad de mantener consultas complejas.

## JPQL (Java Persistence Query Language)

JPQL es un lenguaje de consultas similar a SQL pero que opera sobre el modelo de objetos, no sobre las tablas. Las consultas referencian nombres de entidades y sus atributos en lugar de nombres de tablas y columnas. El ORM traduce automaticamente el JPQL al SQL del dialecto configurado.

```java
// Seleccionar todas las entidades de un tipo
TypedQuery<Producto> query = em.createQuery(
    "SELECT p FROM Producto p", Producto.class);
List<Producto> productos = query.getResultList();
```

### SELECT, WHERE y JOIN

La estructura basica de una consulta JPQL sigue la forma `SELECT ... FROM ... WHERE ... ORDER BY`. A diferencia de SQL nativo, el SELECT es opcional cuando se selecciona la entidad completa.

```java
// Proyeccion: seleccionar atributos especificos
TypedQuery<Object[]> query = em.createQuery(
    "SELECT p.nombre, p.precio FROM Producto p WHERE p.activo = true",
    Object[].class);

// Con parametros nombrados
TypedQuery<Producto> query = em.createQuery(
    "SELECT p FROM Producto p " +
    "WHERE p.categoria = :categoria AND p.precio BETWEEN :min AND :max",
    Producto.class);
query.setParameter("categoria", Categoria.ELECTRONICA);
query.setParameter("min", BigDecimal.valueOf(100));
query.setParameter("max", BigDecimal.valueOf(1000));
```

Los JOINs en JPQL se expresan con la misma sintaxis que SQL, pero referenciando asociaciones del modelo:

```java
// JOIN implicito (navegacion por atributo)
TypedQuery<Pedido> query = em.createQuery(
    "SELECT p FROM Pedido p WHERE p.cliente.email = :email",
    Pedido.class);

// JOIN explicito
TypedQuery<Pedido> query = em.createQuery(
    "SELECT p FROM Pedido p JOIN p.cliente c WHERE c.activo = true",
    Pedido.class);

// LEFT JOIN FETCH para forzar carga de relacion lazy
TypedQuery<Cliente> query = em.createQuery(
    "SELECT c FROM Cliente c LEFT JOIN FETCH c.pedidos WHERE c.id = :id",
    Cliente.class);
```

`JOIN FETCH` es la herramienta clave para resolver el problema N+1. Indica al ORM que cargue la relacion en la misma consulta mediante un JOIN, evitando consultas adicionales al acceder a la propiedad.

### GROUP BY y funciones de agregacion

JPQL soporta las funciones de agregacion estandar: `COUNT`, `SUM`, `AVG`, `MAX`, `MIN`.

```java
TypedQuery<Object[]> query = em.createQuery(
    "SELECT c.nombre, COUNT(p), SUM(p.total) " +
    "FROM Pedido p JOIN p.cliente c " +
    "WHERE p.fecha BETWEEN :inicio AND :fin " +
    "GROUP BY c.nombre " +
    "HAVING COUNT(p) > 5 " +
    "ORDER BY SUM(p.total) DESC",
    Object[].class);
```

Las funciones `COUNT`, `SUM`, `AVG`, etc., se aplican sobre atributos de las entidades. No se pueden usar directamente sobre tablas, solo sobre el modelo de objetos.

### Paginacion

JPA ofrece paginacion nativa mediante `setFirstResult` y `setMaxResults`, independientemente del motor de base de datos. El ORM traduce estas llamadas al dialecto apropiado (LIMIT/OFFSET en PostgreSQL/MySQL, `ROW_NUMBER()` en SQL Server, `ROWNUM` en Oracle).

```java
TypedQuery<Producto> query = em.createQuery("SELECT p FROM Producto p ORDER BY p.id", Producto.class);
query.setFirstResult(20);  // saltar primeros 20
query.setMaxResults(10);   // devolver 10 resultados
List<Producto> pagina = query.getResultList();
```

> La paginacion sin `ORDER BY` no garantiza resultados consistentes entre llamadas. Siempre incluir una clausula `ORDER BY` sobre una columna determinista.

## Named queries (@NamedQuery)

Las consultas JPQL pueden definirse directamente en la entidad mediante `@NamedQuery`, centralizando las consultas y permitiendo su validacion en tiempo de despliegue. El nombre de la named query debe ser unico dentro del contexto de persistencia.

```java
@Entity
@NamedQueries({
    @NamedQuery(name = "Producto.findByCategoria",
                query = "SELECT p FROM Producto p WHERE p.categoria = :categoria"),
    @NamedQuery(name = "Producto.findActivos",
                query = "SELECT p FROM Producto p WHERE p.activo = true " +
                        "ORDER BY p.nombre"),
    @NamedQuery(name = "Producto.countByCategoria",
                query = "SELECT COUNT(p) FROM Producto p GROUP BY p.categoria")
})
public class Producto { ... }

// Uso en codigo
TypedQuery<Producto> query = em.createNamedQuery("Producto.findByCategoria", Producto.class);
query.setParameter("categoria", Categoria.ALIMENTACION);
List<Producto> resultado = query.getResultList();
```

Las named queries se validan al iniciar la aplicacion (cuando se crea el `EntityManagerFactory`), lo que permite detectar errores de sintaxis temprano. Ademas, los proveedores como Hibernate pueden precompilarlas para mejorar el rendimiento.

## Native queries (@Query native=true)

Cuando JPQL no es suficiente (funciones de ventana, consultas especificas del motor, operaciones DDL), JPA permite ejecutar SQL nativo mediante `createNativeQuery`. El resultado puede mapearse a entidades o devolverse como Object[].

```java
// SQL nativo devolviendo entidades
Query query = em.createNativeQuery(
    "SELECT * FROM productos WHERE precio > ?1 ORDER BY nombre LIMIT ?2",
    Producto.class);
query.setParameter(1, BigDecimal.valueOf(100));
query.setParameter(2, 50);
List<Producto> resultado = query.getResultList();

// SQL nativo con proyeccion a DTO
Query query = em.createNativeQuery(
    "SELECT c.nombre AS cliente, COUNT(p.id) AS total_pedidos " +
    "FROM clientes c LEFT JOIN pedidos p ON p.cliente_id = c.id " +
    "GROUP BY c.id, c.nombre " +
    "HAVING COUNT(p.id) > :min",
    "ClienteResumenMapping");  // requiere @SqlResultSetMapping
```

`@SqlResultSetMapping` permite mapear el resultado de una consulta nativa a entidades o DTOs de forma estructurada.

```java
@SqlResultSetMapping(
    name = "ClienteResumenMapping",
    classes = @ConstructorResult(
        targetClass = ClienteResumenDTO.class,
        columns = {
            @ColumnResult(name = "cliente", type = String.class),
            @ColumnResult(name = "total_pedidos", type = Long.class)
        }
    )
)
```

> Las consultas nativas rompen la portabilidad. Si se usan, deben aislarse en repositorios especificos y documentarse las dependencias del motor de BD.

## Criteria API

Criteria API permite construir consultas JPQL programaticamente mediante un builder pattern, proporcionando tipado seguro en tiempo de compilacion. Es especialmente util cuando la consulta se construye dinamicamente segun filtros opcionales.

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Producto> cq = cb.createQuery(Producto.class);
Root<Producto> root = cq.from(Producto.class);

// Construir predicados dinamicamente
List<Predicate> predicates = new ArrayList<>();

if (categoria != null) {
    predicates.add(cb.equal(root.get("categoria"), categoria));
}
if (precioMin != null) {
    predicates.add(cb.greaterThanOrEqualTo(root.get("precio"), precioMin));
}
if (precioMax != null) {
    predicates.add(cb.lessThanOrEqualTo(root.get("precio"), precioMax));
}
if (soloActivos) {
    predicates.add(cb.isTrue(root.get("activo")));
}

cq.where(predicates.toArray(new Predicate[0]));
cq.orderBy(cb.asc(root.get("nombre")));

TypedQuery<Producto> query = em.createQuery(cq);
List<Producto> resultado = query.getResultList();
```

### Ventajas de Criteria API

La principal ventaja es el tipado seguro: si se renombra un atributo de la entidad, el compilador detecta el error. Ademas, las consultas son resistentes a la inyeccion de JPQL (analogo a SQL injection) porque los valores se pasan como parametros.

```java
// Consulta con JOIN usando Criteria API
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Cliente> cq = cb.createQuery(Cliente.class);
Root<Cliente> cliente = cq.from(Cliente.class);
Join<Cliente, Pedido> pedidos = cliente.join("pedidos", JoinType.LEFT);

cq.select(cliente)
  .where(cb.greaterThan(pedidos.get("total"), cb.parameter(BigDecimal.class, "totalMin")))
  .distinct(true)
  .orderBy(cb.asc(cliente.get("nombre")));

TypedQuery<Cliente> query = em.createQuery(cq);
query.setParameter("totalMin", BigDecimal.valueOf(1000));
```

Criteria API tambien soporta subconsultas, funciones de agregacion, agrupacion y todos los constructores de JPQL, aunque la sintaxis builder puede resultar verbosa en comparacion con JPQL textual.

### Metamodel tipado (JPA Metamodel)

Para maximizar el tipado, JPA permite generar un metamodel estatico mediante procesamiento de anotaciones. Por cada entidad `Producto`, se genera una clase `Producto_` con atributos estaticos tipados.

```java
// Generado automaticamente por el procesador de anotaciones
@StaticMetamodel(Producto.class)
public class Producto_ {
    public static volatile SingularAttribute<Producto, Long> id;
    public static volatile SingularAttribute<Producto, String> nombre;
    public static volatile SingularAttribute<Producto, BigDecimal> precio;
    public static volatile SingularAttribute<Producto, Categoria> categoria;
}

// Uso con el metamodel
Root<Producto> root = cq.from(Producto.class);
predicates.add(cb.equal(root.get(Producto_.categoria), categoria));
predicates.add(cb.greaterThanOrEqualTo(root.get(Producto_.precio), precioMin));
```

El metamodel elimina los strings literales en los nombres de atributos, haciendo el codigo completamente refactorizable. Si se renombra `precio` a `precioUnitario`, el compilador marcara todos los lugares donde se use `Producto_.precio`.
