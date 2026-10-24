# Anotaciones basicas

JPA define un conjunto de anotaciones que configuran el mapeo objeto-relacional de forma declarativa, eliminando la necesidad de archivos XML de mapeo. Cada anotacion especifica un aspecto concreto de la correspondencia entre la clase Java y la tabla relacional. Comprenderlas en detalle es requisito indispensable para modelar correctamente cualquier esquema de persistencia.

## @Entity

La anotacion mas basica. Marca una clase como una entidad JPA, es decir, una clase cuyas instancias se almacenan en una tabla de la base de datos. Sin `@Entity`, JPA ignora la clase. La clase debe tener un constructor sin argumentos (public o protected) y debe ser `public`.

```java
@Entity
public class Producto {
    // constructor sin argumentos (obligatorio implicito o explicito)
    public Producto() {}

    @Id
    private Long id;
}
```

Si la clase se llama `Producto`, JPA asume por defecto que la tabla se llama `producto`. Para personalizar el nombre de la tabla se usa `@Table`.

## @Table

Especifica la tabla de la base de datos a la que se mapea la entidad. Es opcional; si se omite, JPA deduce el nombre de la tabla a partir del nombre de la entidad. Se recomienda incluirla siempre para tener control explicito sobre el esquema.

```java
@Entity
@Table(name = "productos", schema = "inventario", catalog = "ecommerce")
public class Producto { ... }
```

Atributos principales:
- `name`: nombre de la tabla (por defecto, nombre de la entidad).
- `schema`: esquema de la base de datos.
- `catalog`: catalogo (menos usado, depende del motor de BD).
- `uniqueConstraints`: permite definir restricciones UNIQUE compuestas a nivel de tabla.

```java
@Table(name = "matriculas",
       uniqueConstraints = {
           @UniqueConstraint(columnNames = {"alumno_id", "curso_id", "anio_academico"})
       })
public class Matricula { ... }
```

## @Id

Marca el atributo que actua como clave primaria de la entidad. Toda entidad JPA debe tener exactamente un atributo anotado con `@Id` (o usar `@EmbeddedId` para claves compuestas). El tipo del atributo debe ser serializable y, para claves simples, suele ser `Long`, `Integer`, `String` o `java.util.UUID`.

```java
@Entity
public class Cliente {
    @Id
    private Long id;
}
```

Para claves primarias compuestas existen dos enfoques:

- `@IdClass`: se define una clase separada que contiene los campos de la clave.
- `@EmbeddedId`: se usa una clase `@Embeddable` como tipo del atributo `@Id`.

```java
// Enfoque con @IdClass
public class PedidoId implements Serializable {
    private Long clienteId;
    private Long numeroPedido;
    // equals() y hashCode() obligatorios
}

@Entity
@IdClass(PedidoId.class)
public class Pedido {
    @Id
    private Long clienteId;
    @Id
    private Long numeroPedido;
}
```

## @GeneratedValue

Define la estrategia de generacion de valores para la clave primaria. Se combina con `@Id` y su atributo `strategy` acepta cuatro valores del enum `GenerationType`:

**AUTO**: el proveedor JPA elige la estrategia mas adecuada segun el motor de base de datos. Para Hibernate, el comportamiento exacto depende del dialecto configurado. Es la opcion por defecto y la mas portable, pero ofrece poco control.

```java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;
```

**IDENTITY**: delega la generacion del ID en la columna auto-incremental de la base de datos. Funciona con `SERIAL` (PostgreSQL), `AUTO_INCREMENT` (MySQL), `IDENTITY` (SQL Server, H2). Es simple pero impide que JPA conozca el ID antes del INSERT, lo que fuerza un `flush()` inmediato en operaciones batch.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

**SEQUENCE**: usa una secuencia de la base de datos. Es la estrategia mas eficiente y recomendada para PostgreSQL, Oracle y DB2. Requiere definir la secuencia con `@SequenceGenerator`, donde se especifican su nombre, el nombre de la secuencia en BD y el tamaño de asignacion (allocationSize).

```java
@Entity
public class Pedido {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "pedido_seq")
    @SequenceGenerator(name = "pedido_seq",
                       sequenceName = "pedidos_id_seq",
                       allocationSize = 50)
    private Long id;
}
```

El `allocationSize` indica cuantos IDs se reservan en memoria de cada vez, reduciendo las llamadas a la secuencia. Debe coincidir con el incremento definido en la secuencia de la BD.

**TABLE**: simula una secuencia mediante una tabla con una fila por secuencia. Es la unica estrategia portable a cualquier base de datos, pero tambien la mas lenta, ya que requiere bloqueos a nivel de fila. Se usa solo cuando no hay soporte nativo de secuencias ni columnas auto-incrementales.

```java
@Id
@GeneratedValue(strategy = GenerationType.TABLE,
                generator = "tabla_generadora")
@TableGenerator(name = "tabla_generadora",
                table = "secuencias_jpa",
                pkColumnName = "nombre_secuencia",
                valueColumnName = "valor_secuencia",
                allocationSize = 50)
private Long id;
```

## @Column

Configura la columna asociada a un atributo. Sin `@Column`, JPA deduce el nombre de la columna a partir del nombre del atributo, siguiendo la estrategia de nomenclatura configurada (por defecto, el mismo nombre). Los atributos mas relevantes son:

| Atributo | Descripcion |
|---|---|
| `name` | Nombre de la columna |
| `nullable` | Si acepta NULL (por defecto true) |
| `length` | Longitud de columna String (solo en DDL, default 255) |
| `precision` | Precision total para BigDecimal |
| `scale` | Escala (decimales) para BigDecimal |
| `unique` | Restriccion UNIQUE (por defecto false) |
| `updatable` | Si la columna se incluye en UPDATE (default true) |
| `insertable` | Si la columna se incluye en INSERT (default true) |
| `columnDefinition` | Definicion SQL explicita de la columna |

```java
@Column(name = "codigo_producto", nullable = false, unique = true, length = 20)
private String codigo;

@Column(precision = 10, scale = 2)
private BigDecimal precio;

@Column(name = "fecha_alta", updatable = false)
private LocalDateTime fechaAlta;
```

> `updatable = false` es util para columnas que se escriben una sola vez (fecha de creacion, usuario creador). `insertable = false` se usa para columnas calculadas o generadas por la BD.

## @Basic

Anotacion opcional que marca un atributo como persistente simple. Se asume por defecto para todos los atributos de la entidad, salvo los anotados con `@Transient`. Se usa cuando se necesita modificar el comportamiento por defecto, como el fetching (LAZY para cargar bajo demanda) o la obligatoriedad.

```java
@Basic(fetch = FetchType.LAZY, optional = false)
private String descripcionLarga;
```

`FetchType.LAZY` para atributos basicos es una peticion: el proveedor puede ignorarlo si el tipo de dato no soporta proxies (como los tipos primitivos o String).

## @Transient

Excluye un atributo de la persistencia. JPA ignora completamente el atributo y no lo mapea a ninguna columna. Es util para atributos calculados, flags internos o propiedades que no deben persistirse.

```java
@Transient
private boolean modificado;

@Transient
private String nombreCompletoCalculado;
```

## @Lob

Marca un atributo como Large Object (LOB), ya sea BLOB (binario) o CLOB (texto largo). JPA mapea automaticamente segun el tipo: `byte[]`, `Byte[]` o `Blob` a `BLOB`; `String`, `char[]`, `Character[]` o `Clob` a `CLOB`. Es recomendable usar `FetchType.LAZY` para no cargar objetos grandes en cada consulta.

```java
@Lob
@Basic(fetch = FetchType.LAZY)
private byte[] imagen;

@Lob
@Basic(fetch = FetchType.LAZY)
private String documentoHtml;
```

## @Enumerated

Mapea un atributo de tipo `enum` a la base de datos. Acepta dos modos:

- `EnumType.ORDINAL` (por defecto): almacena la posicion del enum como entero (0, 1, 2...). Es compacto pero fragil: si se anade un valor al enum en medio, cambian los indices y los datos existentes se corrompen.

- `EnumType.STRING`: almacena el nombre del valor del enum. Ocupa mas espacio pero es resistente a cambios en el orden del enum.

```java
public enum EstadoPedido { CREADO, PAGADO, ENVIADO, ENTREGADO }

@Entity
public class Pedido {
    @Enumerated(EnumType.STRING)  // recomendado
    @Column(length = 20)
    private EstadoPedido estado;
}
```

> **Regla practica**: usar siempre `EnumType.STRING` a menos que la restriccion de espacio sea critica y el enum sea estable.

## @Temporal

JPA 2.x requeria `@Temporal` para mapear tipos `java.util.Date` y `java.util.Calendar`, indicando si se almacena solo `DATE`, solo `TIME` o `TIMESTAMP`. Con JPA 3.x y las nuevas APIs de tiempo de Java 8 (`LocalDate`, `LocalTime`, `LocalDateTime`, `Instant`), `@Temporal` ya no es necesaria para estos tipos: JPA los mapea automaticamente mediante convertidores estandar.

```java
// JPA 2.x (tipos legacy)
@Temporal(TemporalType.TIMESTAMP)
private java.util.Date fechaCreacion;

@Temporal(TemporalType.DATE)
private java.util.Date fechaNacimiento;

// JPA 3.x (Java 8 time)
private LocalDate fechaNacimiento;
private LocalDateTime fechaCreacion;
private Instant instante;
```
