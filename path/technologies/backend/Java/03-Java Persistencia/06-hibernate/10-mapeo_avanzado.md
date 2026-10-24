# Mapeo avanzado

El mapeo objeto-relacional de Hibernate va mucho mas alla de las anotaciones basicas `@Entity`, `@Table` y `@Column`. En escenarios reales aparecen estructuras como tipos embebidos, colecciones y jerarquias de herencia, cada una con sus propias estrategias de mapeo y consideraciones de rendimiento.

## @Embeddable

La anotacion `@Embeddable` permite definir un tipo de objeto que no es una entidad independiente sino que se mapea a las mismas columnas de la tabla de la entidad que lo contiene. Es util para agrupar datos logicamente relacionados sin crear una tabla separada.

```java
@Embeddable
public class Direccion {

    @Column(name = "calle", length = 150)
    private String calle;

    @Column(name = "ciudad", length = 100)
    private String ciudad;

    @Column(name = "codigo_postal", length = 10)
    private String codigoPostal;

    @Column(name = "pais", length = 50)
    private String pais;
}

@Entity
@Table(name = "clientes")
public class Cliente {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column
    private String nombre;

    @Embedded
    private Direccion direccion;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "calle", column = @Column(name = "calle_envio")),
        @AttributeOverride(name = "ciudad", column = @Column(name = "ciudad_envio")),
        @AttributeOverride(name = "codigoPostal", column = @Column(name = "codigo_postal_envio")),
        @AttributeOverride(name = "pais", column = @Column(name = "pais_envio"))
    })
    private Direccion direccionEnvio;
}
```

> La tabla `clientes` contendra todas las columnas de `Direccion` mas las columnas redefinidas para `direccionEnvio`. Los objetos `@Embeddable` no tienen identidad propia ni pueden ser consultados independientemente.

## Collection mapping

Hibernate permite mapear colecciones de tipos basicos o embebidos como atributos de una entidad. Las colecciones se almacenan en tablas separadas (coleccion-owner) que Hibernate crea automaticamente.

### List

```java
@Entity
@Table(name = "articulos")
public class Articulo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ElementCollection
    @CollectionTable(name = "articulos_etiquetas",
        joinColumns = @JoinColumn(name = "articulo_id"))
    @Column(name = "etiqueta")
    @OrderColumn(name = "posicion")
    private List<String> etiquetas;
}
```

`@OrderColumn` asegura que la lista mantenga el orden de insercion. Sin esta anotacion, el orden no esta garantizado. La tabla `articulos_etiquetas` tendra las columnas `articulo_id`, `posicion` y `etiqueta`.

### Set

```java
@ElementCollection
@CollectionTable(name = "articulos_categorias",
    joinColumns = @JoinColumn(name = "articulo_id"))
@Column(name = "categoria")
private Set<String> categorias;
```

Los Sets no permiten duplicados y no requieren `@OrderColumn` porque no tienen orden intrinseco. Si se necesita orden, se puede usar `@OrderBy` o `@SortNatural`.

### Map

```java
@ElementCollection
@CollectionTable(name = "articulo_metadatos",
    joinColumns = @JoinColumn(name = "articulo_id"))
@MapKeyColumn(name = "clave")
@Column(name = "valor")
private Map<String, String> metadatos;
```

Los Map se almacenan con una columna para la clave (`@MapKeyColumn`) y otra para el valor.

> Las colecciones `@ElementCollection` tienen una limitacion importante: no soportan la estrategia de cascada ni el borrado en cascada a nivel de entidad. Si se elimina un elemento de la coleccion, Hibernate borra la fila correspondiente de la tabla de coleccion. Sin embargo, los objetos embebidos de la coleccion no tienen ciclo de vida propio ni soportan lazy loading eficiente.

## Herencia: estrategias de mapeo

JPA define tres estrategias para mapear jerarquias de herencia. Cada una tiene implicaciones distintas sobre el esquema de base de datos, el rendimiento de las consultas y las restricciones del modelo.

### SINGLE_TABLE

Toda la jerarquia se almacena en una unica tabla. Una columna discriminadora (`@DiscriminatorColumn`) identifica el tipo concreto de cada fila.

```java
@Entity
@Table(name = "vehiculos")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "tipo_vehiculo", discriminatorType = DiscriminatorType.STRING)
@DiscriminatorValue("VEHICULO")
public class Vehiculo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column
    private String fabricante;

    @Column
    private Integer anio;
}

@Entity
@DiscriminatorValue("COCHE")
public class Coche extends Vehiculo {

    @Column
    private Integer numeroPuertas;
}

@Entity
@DiscriminatorValue("MOTO")
public class Moto extends Vehiculo {

    @Column
    private Boolean tieneSidecar;
}
```

Ventajas: consultas sin joins, rendimiento optimo. Desventajas: las columnas de subclases deben permitir `null`, no hay restricciones NOT NULL sobre atributos de subclases, la tabla puede crecer con muchas columnas opcionales.

> `SINGLE_TABLE` es la estrategia mas rapida para consultas polimorficas (consultas que buscan todas las subclases) porque no requiere joins. Sin embargo, la falta de restricciones NOT NULL puede provocar inconsistencias si no se valida a nivel de aplicacion.

### JOINED

Cada clase de la jerarquia tiene su propia tabla. La tabla de la subclase tiene solo las columnas especificas y un JOIN con la tabla de la superclase.

```java
@Entity
@Table(name = "vehiculos")
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "tipo_vehiculo", discriminatorType = DiscriminatorType.STRING)
public class Vehiculo {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String fabricante;
    private Integer anio;
}

@Entity
@Table(name = "coches")
@PrimaryKeyJoinColumn(name = "vehiculo_id")
public class Coche extends Vehiculo {
    private Integer numeroPuertas;
}
```

Ventajas: normalizacion completa, cada columna NOT NULL en su tabla, sin columnas opcionales. Desventajas: las consultas requieren joins entre tablas, lo que penaliza el rendimiento en jerarquias profundas.

> La columna discriminadora es opcional en `JOINED` (no se necesita para determinar el tipo porque Hibernate puede deducirlo por la presencia o ausencia de filas en las tablas hijas), pero se recomienda incluirla porque mejora el rendimiento de las consultas polimorficas.

### TABLE_PER_CLASS

Cada clase concreta tiene su propia tabla independiente que incluye todas las columnas de la superclase mas las suyas propias. No hay tabla para la superclases si la superclase es abstracta.

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Vehiculo {
    @Id
    @GeneratedValue(generator = "vehiculo_seq")
    @GenericGenerator(name = "vehiculo_seq", strategy = "org.hibernate.id.enhanced.SequenceStyleGenerator")
    private Long id;
    private String fabricante;
    private Integer anio;
}

@Entity
@Table(name = "coches")
public class Coche extends Vehiculo {
    private Integer numeroPuertas;
}
```

Ventajas: no hay joins, cada tabla es independiente. Desventajas: las consultas polimorficas requieren UNION entre todas las tablas, la generacion de IDs debe usar una secuencia compartida porque los IDs deben ser unicos entre todas las tablas.

> `TABLE_PER_CLASS` no soporta la generacion de identidad con `IDENTITY` porque los IDs deben ser unicos globalmente, no por tabla. Se recomienda usar `SEQUENCE` o `TABLE` en esta estrategia.

## Discriminator columns

La columna discriminadora se usa en `SINGLE_TABLE` y opcionalmente en `JOINED` para identificar el tipo concreto de cada entidad en una jerarquia de herencia.

```java
@Entity
@DiscriminatorColumn(
    name = "tipo",
    discriminatorType = DiscriminatorType.STRING,
    length = 20
)
@DiscriminatorValue("BASE")
public class BaseEntity {
    // ...
}
```

Los tipos de discriminador disponibles son `STRING` (por defecto), `CHAR` e `INTEGER`. Hibernate puede tambien usar formulas SQL como discriminador mediante `@DiscriminatorFormula`:

```java
@DiscriminatorFormula("CASE WHEN tipo = 'PREMIUM' THEN 1 ELSE 0 END")
```

> `@DiscriminatorFormula` es util cuando la logica de discriminacion no puede expresarse con una simple columna, por ejemplo cuando se determina el tipo en funcion de multiples columnas o condiciones.
