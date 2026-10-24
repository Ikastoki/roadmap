# Patrones de persistencia

La capa de persistencia de una aplicacion Java puede organizarse siguiendo distintos patrones arquitectonicos, cada uno con implicaciones directas sobre el acoplamiento, la testabilidad y la flexibilidad del sistema. Estos patrones no son exclusivos de un ORM concreto, aunque ciertos frameworks los implementan de forma nativa. Martin Fowler los catalogo en su obra *Patterns of Enterprise Application Architecture* como las soluciones recurrentes a los problemas de persistencia objeto-relacional.

## Active Record

El patron Active Record consiste en que cada entidad encapsula tanto los datos como la logica de acceso a base de datos. Un objeto `Producto` no solo contiene `id`, `nombre` y `precio`, sino que tambien ofrece metodos como `save()`, `delete()` y `findById()`. La entidad es activa porque conoce como persistirse a si misma.

Active Record es el patron dominante en Rails (Ruby on Rails) y en frameworks como Laravel (PHP). En el ecosistema Java no es tan comun, aunque ciertos ORMs ligeros y microframeworks lo implementan. Su principal ventaja es la simplicidad: todo lo relativo a una entidad esta en un mismo lugar. El inconveniente es que viola el principio de responsabilidad unica y dificulta las pruebas unitarias, ya que la logica de negocio queda acoplada al mecanismo de persistencia.

```java
public class Producto {
    private Long id;
    private String nombre;
    private BigDecimal precio;

    // logica de negocio
    public void aplicarDescuento(BigDecimal porcentaje) {
        this.precio = this.precio.multiply(
            BigDecimal.ONE.subtract(porcentaje.divide(BigDecimal.valueOf(100))));
    }

    // persistencia (Active Record)
    public void save() { /* INSERT o UPDATE segun estado */ }
    public void delete() { /* DELETE */ }
    public static Producto findById(Long id) { /* SELECT */ }
}
```

## Data Mapper

Data Mapper es el patron opuesto a Active Record. Las entidades son objetos puramente de dominio (POJOs) que desconocen completamente el mecanismo de persistencia. Un objeto separado, el mapper, se encarga de transferir datos entre la base de datos y las entidades. Este mapper contiene toda la logica de transformacion y es quien ejecuta las sentencias SQL.

Hibernate y JPA implementan Data Mapper: el `EntityManager` actua como el mapper central que sincroniza entidades con la base de datos. Las entidades no tienen metodos `save()` ni `find()`; simplemente son objetos Java que el EntityManager gestiona. Esto permite que las entidades se centren exclusivamente en la logica de negocio y sean facilmente testables.

```java
// Entidad pura (ignorante de persistencia)
public class Producto {
    private Long id;
    private String nombre;
    private BigDecimal precio;

    public void aplicarDescuento(BigDecimal porcentaje) {
        this.precio = ...;
    }
}

// El mapper es externo (EntityManager)
// producto = em.find(Producto.class, 1L);
// em.persist(producto);
```

La separacion que introduce Data Mapper es arquitectonicamente mas limpia, pero requiere mas codigo infraestructural. JPA simplifica esta capa gracias a las anotaciones de mapeo, pero el patron subyacente sigue siendo el mismo.

## Table Data Gateway

Table Data Gateway actua como una puerta de enlace a una tabla de la base de datos. A diferencia de Active Record, el gateway no esta acoplado a una entidad concreta, sino que expone metodos para operar sobre la tabla: `find(id)`, `findAll()`, `insert(obj)`, `update(obj)`, `delete(id)`. El gateway devuelve objetos simples (DTOs o Maps) en lugar de entidades de dominio.

Este patron es util cuando no se desea un modelo de dominio completo o cuando la aplicacion sigue un enfoque tabular. Spring Data JDBC y JdbcTemplate permiten implementar Table Data Gateway de forma natural.

```java
public class ProductoGateway {
    private final JdbcTemplate jdbcTemplate;

    public ProductoGateway(DataSource dataSource) {
        this.jdbcTemplate = new JdbcTemplate(dataSource);
    }

    public ProductoDTO findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT id, nombre, precio FROM productos WHERE id = ?",
            (rs, row) -> new ProductoDTO(
                rs.getLong("id"),
                rs.getString("nombre"),
                rs.getBigDecimal("precio")
            ), id);
    }

    public void insert(ProductoDTO producto) {
        jdbcTemplate.update(
            "INSERT INTO productos (nombre, precio) VALUES (?, ?)",
            producto.getNombre(), producto.getPrecio());
    }
}
```

## Repository

El patron Repository abstrae el almacenamiento subyacente tras una interfaz con nombre de dominio. A diferencia de Table Data Gateway (que se organiza por tablas), el repositorio se organiza por conceptos de dominio. `ProductoRepository` no expone metodos SQL, sino operaciones del lenguaje del dominio: `buscarPorNombre()`, `buscarConStockBajo()`, `productosMasVendidos()`.

El repositorio media entre la capa de dominio y la capa de persistencia, dando la ilusion de que todas las entidades estan en memoria. Spring Data JPA eleva este patron a su maxima expresion: basta con definir una interfaz que extiende `JpaRepository` y Spring genera automaticamente la implementacion.

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {
    List<Producto> findByNombreContainingIgnoreCase(String nombre);
    List<Producto> findByPrecioBetween(BigDecimal min, BigDecimal max);
    @Query("SELECT p FROM Producto p WHERE p.stock < :minimo")
    List<Producto> buscarConStockBajo(@Param("minimo") Integer stockMinimo);
}
```

La ventaja fundamental del repositorio es que permite cambiar la implementacion de persistencia sin afectar al codigo que lo consume. Una interfaz `ProductoRepository` puede tener una implementacion con JPA, otra con MongoDB y otra en memoria para pruebas, todas intercambiables.

## Unit of Work

Unit of Work (Unidad de Trabajo) mantiene un registro de todos los cambios realizados sobre las entidades durante una transaccion de negocio. Al finalizar, determina que operaciones deben ejecutarse contra la base de datos y en que orden para persistir todos los cambios de forma atomica.

El `EntityManager` de JPA es una implementacion del patron Unit of Work. Cuando se llama a `persist()`, `merge()` o `remove()`, los cambios no se reflejan inmediatamente en la base de datos. En su lugar, el EntityManager mantiene un mapa de entidades nuevas, modificadas y eliminadas. Al hacer `flush()` (implicito en `commit()`), el ORM genera automaticamente las sentencias INSERT, UPDATE y DELETE en el orden correcto, respetando las claves ajenas.

```java
// Todo esto se ejecuta en una unica transaccion
entityManager.persist(pedido);          // marca como nuevo
entityManager.merge(cliente);           // marca como modificado (si ya existe)
entityManager.remove(lineaAntigua);     // marca como eliminado

// Al hacer commit(), el ORM decide el orden:
// 1. INSERT pedido
// 2. UPDATE cliente
// 3. DELETE linea_pedido WHERE id = ...
```

Sin Unit of Work, el desarrollador tendria que gestionar manualmente el orden de las operaciones y garantizar la atomicidad. El patron simplifica drasticamente la logica transaccional.

## Identity Map

Identity Map garantiza que cada fila de la base de datos se cargue una unica vez en memoria por sesion. Si se solicita la misma entidad dos veces, el ORM devuelve la misma instancia Java (misma referencia de memoria), no dos copias. Esto evita inconsistencias donde una misma fila se modifica en dos objetos distintos.

El EntityManager de JPA implementa Identity Map mediante su cache de primer nivel (first-level cache). Cuando se invoca `find(Producto.class, 1L)` dos veces, solo la primera llamada ejecuta una consulta SQL; la segunda devuelve la entidad directamente de la cache.

```java
Producto p1 = entityManager.find(Producto.class, 1L);
// SQL: SELECT * FROM productos WHERE id = 1

Producto p2 = entityManager.find(Producto.class, 1L);
// No ejecuta SQL, devuelve la misma instancia que p1

System.out.println(p1 == p2); // true
```

Identity Map es esencial para mantener la consistencia dentro de una transaccion larga y para evitar la propagacion de cambios no deseados entre distintas partes del codigo que acceden a la misma entidad.
