# Mapeo de tipos

Uno de los cometidos principales de JPA es traducir los tipos del lenguaje Java a tipos SQL y viceversa. Esta traduccion es automatica para la mayoria de los tipos basicos, pero ciertos tipos de dominio requieren configuracion explicita mediante anotaciones o convertidores. Un mapeo de tipos incorrecto es fuente de errores silenciosos que se manifiestan como datos truncados, excepciones de casting o incompatibilidades entre entornos de desarrollo y produccion.

## Tipos basicos

JPA mapea automaticamente los tipos Java mas comunes a sus equivalentes SQL, siguiendo la tabla estandar definida por la especificacion SQL y adaptada segun el dialecto de cada base de datos.

| Tipo Java | Tipo SQL (tipico) | Notas |
|---|---|---|
| `String` | `VARCHAR(255)` | Longitud configurable con `@Column(length=N)` |
| `int`, `Integer` | `INTEGER` | |
| `long`, `Long` | `BIGINT` | |
| `short`, `Short` | `SMALLINT` | |
| `byte`, `Byte` | `TINYINT` | |
| `float`, `Float` | `FLOAT` / `REAL` | Puede perder precision |
| `double`, `Double` | `DOUBLE` | |
| `boolean`, `Boolean` | `BOOLEAN` / `BIT` / `TINYINT(1)` | Depende del dialecto |
| `BigDecimal` | `NUMERIC(p, s)` | Usar `@Column(precision, scale)` |
| `BigInteger` | `BIGINT` / `NUMERIC(p)` | |
| `byte[]` | `VARBINARY` / `BLOB` | |

```java
@Entity
@Table(name = "productos")
public class Producto {
    @Id
    private Long id;

    @Column(length = 100, nullable = false)
    private String nombre;

    @Column(precision = 12, scale = 4)
    private BigDecimal precioUnitario;

    private boolean activo;

    private Integer stockMinimo;  // permite null si no esta definido
}
```

> Los tipos primitivos (`int`, `long`, `boolean`) se mapean como columnas NOT NULL. Los tipos wrapper (`Integer`, `Long`, `Boolean`) permiten NULL. Elegir entre uno u otro tiene implicaciones semanticas: un `BigDecimal` a null es "precio no definido"; un `boolean` a false es "no activo".

## Fechas y tiempos (Date/Time)

JPA 2.x solo soportaba nativamente `java.util.Date`, `java.util.Calendar` y `java.sql.Date/Time/Timestamp`, requiriendo `@Temporal` para especificar la precision. A partir de JPA 3.x (Jakarta Persistence 3.0), el soporte de las APIs de tiempo de Java 8 es nativo mediante convertidores automaticos.

```java
@Entity
public class Evento {
    @Id
    private Long id;

    // Java 8 time - soporte nativo en JPA 3.x
    private LocalDate fechaEvento;       // DATE
    private LocalTime horaInicio;        // TIME
    private LocalDateTime fechaCreacion;  // TIMESTAMP
    private Instant instante;             // TIMESTAMP
    private Duration duracion;            // BIGINT (nanos)
    private Period periodo;               // VARCHAR

    // Legacy - requieren @Temporal
    @Temporal(TemporalType.DATE)
    private java.util.Date fechaSolo;

    @Temporal(TemporalType.TIMESTAMP)
    private java.util.Calendar calendar;
}
```

Los tipos `LocalDate`, `LocalTime`, `LocalDateTime` e `Instant` se almacenan en columnas `DATE`, `TIME`, `TIMESTAMP` y `TIMESTAMP` respectivamente. La precision real depende del motor de base de datos: PostgreSQL almacena microsegundos en `TIMESTAMP`, MySQL puede almacenar hasta 6 digitos fraccionales si se define `TIMESTAMP(6)`.

> `Instant` se mapea a `TIMESTAMP WITH TIMEZONE` en bases de datos que lo soportan (PostgreSQL). En caso contrario, se almacena como `TIMESTAMP` asumiendo UTC. Es recomendable almacenar siempre las fechas en UTC y convertir a la zona horaria local en la capa de presentacion.

## @Enumerated (STRING vs ORDINAL)

El mapeo de enums merece un tratamiento aparte por las implicaciones de integridad de datos. Un enum puede persistirse como su nombre (STRING) o como su posicion ordinal (ORDINAL).

```java
public enum Categoria {
    ELECTRONICA, ROPA, ALIMENTACION, HOGAR, LIBROS
}

@Entity
public class Articulo {
    // Recomendado: resistente a cambios de orden
    @Enumerated(EnumType.STRING)
    @Column(length = 20, nullable = false)
    private Categoria categoria;

    // Fragil: si se anade una categoria entre ELECTRONICA y ROPA,
    // los registros existentes quedan corruptos
    @Enumerated(EnumType.ORDINAL)
    private Categoria categoriaLegacy;
}
```

Con `ORDINAL`, si el enum actual es `{ELECTRONICA=0, ROPA=1, ALIMENTACION=2}` y se inserta `DEPORTES` entre `ROPAS` y `ALIMENTACION`, el nuevo orden `{ELECTRONICA=0, ROPA=1, DEPORTES=2, ALIMENTACION=3}` cambia la semantica de todos los registros con valor 2 o superior.

**Consejo**: usar siempre `EnumType.STRING` con `@Column(length=N)` y validar que la longitud sea suficiente para el valor mas largo del enum.

## @Embeddable y @Embedded

No todas las columnas de una tabla merecen ser entidades independientes. Un valor de domicilio (calle, ciudad, codigoPostal) son atributos logicamente agrupados pero sin identidad propia. `@Embeddable` permite definir una clase reutilizable cuyas columnas se incluyen en la tabla de la entidad que la contiene.

```java
@Embeddable
public class Direccion {
    @Column(length = 100, nullable = false)
    private String calle;

    @Column(length = 50, nullable = false)
    private String ciudad;

    @Column(length = 10)
    private String codigoPostal;

    @Column(length = 50)
    private String pais;
}

@Entity
@Table(name = "clientes")
public class Cliente {
    @Id
    private Long id;

    @Embedded
    private Direccion direccion;  // las columnas se crean en la tabla "clientes"

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "calle", column = @Column(name = "calle_facturacion")),
        @AttributeOverride(name = "ciudad", column = @Column(name = "ciudad_facturacion"))
    })
    private Direccion direccionFacturacion;
}
```

Las columnas de `Direccion` se incluyen directamente en la tabla `clientes` (`calle`, `ciudad`, `codigoPostal`, `pais`). Si una entidad necesita dos direcciones, `@AttributeOverrides` permite renombrar las columnas para evitar conflictos.

## @ElementCollection y @CollectionTable

Cuando una entidad tiene una coleccion de valores simples o embeddables que no merecen una entidad propia (telefonos, emails, etiquetas), `@ElementCollection` mapea la coleccion a una tabla secundaria sin necesidad de crear una entidad separada.

```java
@Entity
@Table(name = "usuarios")
public class Usuario {
    @Id
    private Long id;

    @ElementCollection
    @CollectionTable(
        name = "usuario_telefonos",
        joinColumns = @JoinColumn(name = "usuario_id")
    )
    @Column(name = "telefono")
    private List<String> telefonos;

    @ElementCollection
    @CollectionTable(
        name = "usuario_direcciones",
        joinColumns = @JoinColumn(name = "usuario_id")
    )
    private List<Direccion> direccionesEnvio;
}
```

Las colecciones `@ElementCollection` se cargan por defecto en modo LAZY. La tabla generada (`usuario_telefonos`) tiene una columna `usuario_id` (clave ajena) y una columna `telefono` para el valor. Para colecciones de `@Embeddable`, se anaden todas las columnas del embeddable.

> Las colecciones `@ElementCollection` no tienen clave primaria propia. Si se necesita eliminar un elemento individual, el ORM suele eliminar todos los registros de la tabla asociada y reinsertar los que quedan. Para colecciones grandes, es mejor modelarlas como entidades reales.
