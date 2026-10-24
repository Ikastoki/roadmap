# Fetch strategies

Las estrategias de carga (fetch strategies) determinan como y cuando Hibernate recupera las entidades relacionadas desde la base de datos. Una configuracion inadecuada puede provocar graves problemas de rendimiento, desde el temido N+1 hasta saturaciones de memoria por carga innecesaria de datos. Comprender las diferencias entre EAGER y LAZY, asi como las distintas estrategias de `@Fetch`, es esencial para disenar una capa de persistencia eficiente.

## EAGER vs LAZY

`FetchType.EAGER` indica a Hibernate que la relacion debe cargarse inmediatamente cuando se carga la entidad propietaria. `FetchType.LAZY` difiere la carga hasta que se accede explicitamente a la propiedad.

```java
@Entity
public class Usuario {
    @OneToMany(mappedBy = "usuario", fetch = FetchType.LAZY)
    private List<Pedido> pedidos;

    @ManyToOne(fetch = FetchType.EAGER)
    private Direccion direccion;
}
```

Con EAGER, Hibernate ejecuta consultas adicionales (tipicamente JOINs) en el momento de cargar la entidad principal. Con LAZY, la propiedad es un proxy que solo se materializa cuando se invoca un metodo sobre ella.

La recomendacion generalmente aceptada es utilizar `FetchType.LAZY` para todas las relaciones. Las razones son contundentes:

- EAGER fuerza la carga de datos que pueden no ser necesarios para el caso de uso actual
- EAGER en multiples relaciones puede generar productos cartesianos o multiples consultas adicionales
- EAGER no se puede desactivar en tiempo de ejecucion; LAZY se puede activar con `JOIN FETCH` o `@EntityGraph` cuando se necesita

> El valor por defecto de `@ManyToOne` y `@OneToOne` es EAGER en JPA. Es recomendable sobreescribirlo explicitamente a LAZY en todas las relaciones para evitar comportamientos inesperados.

## @Fetch

La anotacion `@Fetch` de Hibernate (org.hibernate.annotations.Fetch) define como se ejecuta la carga de una relacion LAZY. Proporciona tres modos:

`@Fetch(FetchMode.JOIN)` fuerza Hibernate a cargar la relacion mediante un JOIN, incluso si el fetch type es LAZY. Su efecto es similar a `JOIN FETCH` en JPQL:

```java
@Entity
public class Usuario {
    @OneToMany(mappedBy = "usuario", fetch = FetchType.LAZY)
    @Fetch(FetchMode.JOIN)
    private List<Pedido> pedidos;
}
```

`FetchMode.JOIN` es una directiva global que afecta a todas las consultas que cargan la entidad. Esto puede ser problematico porque fuerza la carga incluso en casos donde no se necesita.

`@Fetch(FetchMode.SELECT)` ejecuta una consulta adicional por cada entidad principal cuando se accede a la relacion. Es el modo por defecto para colecciones LAZY sin `@BatchSize`:

```java
@OneToMany(mappedBy = "usuario", fetch = FetchType.LAZY)
@Fetch(FetchMode.SELECT)
private List<Pedido> pedidos;
```

Con `FetchMode.SELECT`, acceder a los pedidos de un usuario concreto ejecuta `SELECT * FROM pedidos WHERE usuario_id = ?`. Si se itera sobre 100 usuarios, se ejecutan 100 consultas adicionales (problema N+1).

`@Fetch(FetchMode.SUBSELECT)` carga la relacion para todas las entidades principales mediante una subconsulta:

```java
@OneToMany(mappedBy = "usuario", fetch = FetchType.LAZY)
@Fetch(FetchMode.SUBSELECT)
private List<Pedido> pedidos;
```

La consulta generada es:

```sql
SELECT * FROM pedidos WHERE usuario_id IN (SELECT id FROM usuarios)
```

`FetchMode.SUBSELECT` ejecuta dos consultas: una para las entidades principales y otra para la relacion, independientemente de cuantas entidades principales haya. Es eficiente cuando se cargan colecciones para multiples entidades, pero puede ser contraproducente si la subconsulta es costosa o si solo se necesita la relacion para una o dos entidades.

## @BatchSize combinado con @Fetch

`@BatchSize` suaviza el problema N+1 agrupando las consultas de `FetchMode.SELECT` en lotes:

```java
@OneToMany(mappedBy = "usuario", fetch = FetchType.LAZY)
@Fetch(FetchMode.SELECT)
@BatchSize(size = 20)
private List<Pedido> pedidos;
```

Con 100 usuarios, se ejecutan 5 consultas de 20 usuarios cada una en lugar de 100 consultas individuales:

```sql
SELECT * FROM pedidos WHERE usuario_id IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
```

## @BatchSize a nivel de clase

`@BatchSize` tambien puede aplicarse a nivel de clase para controlar como se cargan los proxies de la entidad:

```java
@Entity
@BatchSize(size = 20)
public class Usuario {
    // @ManyToOne(fetch = FetchType.LAZY) a otra entidad cargara 20 proxies por consulta
}
```

Esto es util cuando se accede a una relacion `@ManyToOne` LAZY desde multiples entidades en un bucle.

## Cuando usar cada estrategia

La eleccion de estrategia de carga depende del patron de acceso a datos:

`FetchMode.JOIN` es adecuado cuando la relacion siempre se necesita junto con la entidad principal. Es equivalente a `JOIN FETCH` global, pero debe usarse con moderacion para evitar productos cartesianos.

`FetchMode.SELECT` con `@BatchSize` es una buena opcion por defecto para colecciones. Proporciona carga LAZY con rendimiento predecible cuando se accede a la relacion.

`FetchMode.SUBSELECT` es util cuando se necesita la relacion para todas las entidades de una consulta y la consulta principal tiene filtros selectivos. Es eficiente con conjuntos pequenos de entidades principales.

`FetchMode.JOIN` debe evitarse en relaciones `@OneToMany` cuando se combina con paginacion, porque Hibernate realiza la paginacion en memoria despues del JOIN, no en la base de datos.

```java
// PELIGRO: Paginacion en memoria con FetchMode.JOIN en @OneToMany
@Entity
public class Categoria {
    @OneToMany(mappedBy = "categoria", fetch = FetchType.LAZY)
    @Fetch(FetchMode.JOIN)
    private List<Producto> productos;
}

// findAll(Pageable) aplica LIMIT/OFFSET en memoria despues del JOIN
// Esto puede devolver menos resultados de los esperados
```

## Resumen de recomendaciones

- Todas las relaciones deben ser `FetchType.LAZY`
- Usar `@EntityGraph` o `JOIN FETCH` en las consultas donde se necesite carga temprana
- Configurar `@BatchSize` globalmente o por entidad como red de seguridad
- Evitar `FetchMode.JOIN` a nivel de anotacion; usar `JOIN FETCH` en consultas especificas
- Preferir DTOs y proyecciones para consultas de solo lectura
