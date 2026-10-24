# N+1 Problem

El problema N+1 es una de las anomalias de rendimiento mas comunes y perniciosas en aplicaciones que utilizan JPA/Hibernate. Se produce cuando, para recuperar una coleccion de N entidades principales, el ORM ejecuta 1 consulta para obtener las entidades principales y N consultas adicionales para obtener las relaciones de cada una, para un total de N+1 consultas. Este comportamiento, aparentemente inofensivo con conjuntos pequenos, se convierte en un problema grave a medida que N crece.

## Causa del problema

El problema N+1 surge tipicamente cuando una relacion `@OneToMany` o `@ManyToMany` se configura con fetch LAZY y se accede a la coleccion fuera del contexto de la consulta original.

```java
@Entity
public class Categoria {
    @Id
    private Long id;
    private String nombre;

    @OneToMany(mappedBy = "categoria", fetch = FetchType.LAZY)
    private List<Producto> productos;
}

@Entity
public class Producto {
    @Id
    private Long id;
    private String nombre;

    @ManyToOne
    private Categoria categoria;
}
```

```java
public List<Categoria> obtenerTodasConProductos() {
    List<Categoria> categorias = categoriaRepository.findAll();
    // Hasta aqui solo 1 consulta: SELECT * FROM categorias
    for (Categoria c : categorias) {
        int size = c.getProductos().size();
        // Aqui se ejecuta 1 SELECT por cada categoria:
        // SELECT * FROM productos WHERE categoria_id = ?
    }
    return categorias;
}
```

Con 100 categorias, la ejecucion genera 101 consultas SQL. Si cada producto tiene mas relaciones anidadas, el problema se multiplica exponencialmente.

## Solucion con JOIN FETCH

`JOIN FETCH` es una clausula JPQL que instruye a Hibernate para cargar la relacion en la misma consulta mediante un JOIN:

```java
@Query("SELECT c FROM Categoria c JOIN FETCH c.productos")
List<Categoria> findAllWithProductos();
```

La consulta SQL generada es:

```sql
SELECT c.*, p.* FROM categorias c
INNER JOIN productos p ON p.categoria_id = c.id
```

Con `JOIN FETCH` se ejecuta una unica consulta independientemente del numero de categorias. Sin embargo, hay que tener en cuenta que `JOIN FETCH` puede generar duplicados en el resultado (una categoria aparece tantas veces como productos tenga), y que no es compatible con paginacion en algunos dialectos.

```java
@Query("SELECT DISTINCT c FROM Categoria c JOIN FETCH c.productos")
List<Categoria> findAllWithProductos();
```

> `JOIN FETCH` solo funciona para una relacion por consulta con JPA estandar. Si se necesitan cargar multiples relaciones, Hibernate permite multiples `JOIN FETCH` pero puede generar productos cartesianos.

## Solucion con @EntityGraph

`@EntityGraph` permite definir grafos de entidades que especifican que relaciones deben cargarse de forma eager en una consulta determinada, sin modificar la configuracion de fetch de la entidad.

```java
@Entity
@NamedEntityGraph(
    name = "Categoria.productos",
    attributeNodes = @NamedAttributeNode("productos")
)
public class Categoria {
    // ...
}
```

```java
public interface CategoriaRepository extends JpaRepository<Categoria, Long> {

    @EntityGraph("Categoria.productos")
    List<Categoria> findAll();
}
```

`@EntityGraph` tambien permite definir grafos ad-hoc directamente en el metodo del repository:

```java
public interface CategoriaRepository extends JpaRepository<Categoria, Long> {

    @EntityGraph(attributePaths = {"productos", "productos.proveedor"})
    List<Categoria> findAll();
}
```

`@EntityGraph` genera un `LEFT JOIN FETCH` para cada relacion especificada. A diferencia de `JOIN FETCH` manual, `@EntityGraph` es declarativo y se integra con la query derivation.

## Solucion con @BatchSize

`@BatchSize` no elimina el problema N+1, sino que lo reduce agrupando las consultas adicionales en lotes. Configurado en la entidad, define cuantas relaciones se cargan por cada consulta adicional.

```java
@Entity
public class Categoria {
    @Id
    private Long id;
    private String nombre;

    @OneToMany(mappedBy = "categoria", fetch = FetchType.LAZY)
    @BatchSize(size = 10)
    private List<Producto> productos;
}
```

Con `@BatchSize(size = 10)`, al acceder a los productos de 100 categorias, Hibernate ejecuta 1 consulta para las categorias y 10 consultas para los productos (cada una cargando los productos de 10 categorias), para un total de 11 consultas en lugar de 101.

```sql
-- Primera consulta (productos de las primeras 10 categorias)
SELECT * FROM productos WHERE categoria_id IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
```

`@BatchSize` es una solucion pragmatica cuando no es posible usar `JOIN FETCH` o `@EntityGraph`, especialmente en consultas con paginacion donde el JOIN FETCH no es viable.

## Solucion con subqueries

Hibernate tambien soporta la anotacion `@Fetch(FetchMode.SUBSELECT)`, que carga las colecciones mediante una subconsulta:

```java
@Entity
public class Categoria {
    @Id
    private Long id;
    private String nombre;

    @OneToMany(mappedBy = "categoria", fetch = FetchType.LAZY)
    @Fetch(FetchMode.SUBSELECT)
    private List<Producto> productos;
}
```

La consulta generada es:

```sql
SELECT * FROM productos WHERE categoria_id IN (SELECT id FROM categorias)
```

`FetchMode.SUBSELECT` ejecuta dos consultas: una para las categorias y una para todos los productos de esas categorias. Es eficiente cuando la consulta principal tiene filtros y solo se carga un subconjunto de entidades, pero puede ser contraproducente si la subconsulta es compleja o si la tabla de productos es muy grande.

## Solucion con DTOs y proyecciones

La solucion mas radical y eficiente para el problema N+1 es evitar la carga de relaciones por completo, utilizando proyecciones o DTOs que contengan exactamente los datos necesarios:

```java
@Query("SELECT new com.ejemplo.dto.CategoriaConCount(c.id, c.nombre, " +
       "COUNT(p.id)) FROM Categoria c LEFT JOIN c.productos p GROUP BY c.id, c.nombre")
List<CategoriaConCount> findAllWithProductCount();
```

Este enfoque elimina cualquier consulta adicional al transferir la responsabilidad de agregacion a la base de datos, que esta altamente optimizada para este tipo de operaciones.

## Deteccion del problema N+1

Hibernate proporciona una propiedad para registrar todas las consultas SQL ejecutadas:

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

Herramientas como `hibernate-statistics` o la integracion con `datasource-proxy` permiten contar el numero de consultas ejecutadas y detectar patrones N+1 durante el desarrollo:

```xml
<dependency>
    <groupId>net.ttddyy</groupId>
    <artifactId>datasource-proxy</artifactId>
    <scope>test</scope>
</dependency>
```

Con esta dependencia, se puede contar exactamente cuantas consultas se ejecutan en cada operacion y detectar visualmente el patron N+1 en los logs.
