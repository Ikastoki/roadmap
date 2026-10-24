# Entity y JPA

Una entidad JPA es un objeto que representa un registro en una tabla de la base de datos. Cada instancia de la entidad se corresponde con una fila, y cada campo de la clase con una columna. La entidad posee un identificador unico que la distingue de cualquier otra instancia, incluso si el resto de sus atributos son identicos. Esta identidad es la propiedad fundamental que separa a una entidad de un simple valor: dos objetos `Persona` con el mismo `nombre` y `edad` pero distinto `id` son dos personas diferentes.

La distincion entre una Entity JPA y un Domain Object es sutil pero critica. La Entity responde a la estructura de la base de datos: sus campos reflejan columnas, sus relaciones reflejan claves foraneas, y sus anotaciones describen el mapeo fisico. El Domain Object, en cambio, responde al lenguaje del negocio: sus metodos implementan reglas, sus nombres usan el vocabulario del dominio, y su estructura optimiza la expresividad, no el esquema de tablas. Una Entity puede tener campos tecnicos (un `version` para optimistic locking, un `createdAt` de auditoria) que el modelo de dominio no necesita ni debe conocer. A la inversa, un Domain Object puede contener metodos de validacion o calculo que la Entity no deberia tener, porque la persistencia no es responsable de la logica de negocio.

## @Entity y @Table

`@Entity` marca una clase como una entidad JPA gestionable por el `EntityManager`. Sin esta anotacion, JPA ignora la clase. `@Table` especifica el nombre de la tabla en la base de datos; si se omite, JPA usa el nombre de la clase.

```java
@Entity
@Table(name = "usuarios")
public class Usuario {
    // ...
}
```

> `@Table` admite atributos adicionales como `schema`, `catalog`, y `uniqueConstraints` para definir constraints a nivel de tabla. El esquema es util en bases de datos que organizan tablas en esquemas logicos (PostgreSQL, SQL Server).

## @Id y @GeneratedValue

Toda entidad JPA debe tener un campo marcado con `@Id` que actua como clave primaria. `@GeneratedValue` define la estrategia de generacion del identificador.

```java
@Entity
public class Usuario {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

Las estrategias de generacion son:

- **IDENTITY**: la base de datos genera el valor automaticamente (auto-increment en MySQL, `SERIAL` en PostgreSQL). Es la mas simple pero impide que JPA conozca el ID antes del `INSERT`.
- **SEQUENCE**: usa una secuencia de base de datos (tipico en Oracle, PostgreSQL). Es la mas eficiente para insercion en lotes porque JPA puede preasignar IDs.
- **TABLE**: simula una secuencia mediante una tabla. Portable entre bases de datos, pero con peor rendimiento.
- **AUTO**: JPA elige la estrategia segun el dialecto de base de datos. Es la opcion por defecto pero puede dar sorpresas al cambiar de base de datos.

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq_usuario")
@SequenceGenerator(name = "seq_usuario", sequenceName = "seq_usuario", allocationSize = 1)
private Long id;
```

El `allocationSize` del `@SequenceGenerator` controla cuantos IDs se preasignan en memoria. Un valor de 50 (el defecto) reduce las llamadas a la base de datos pero puede dejar huecos en la secuencia si la aplicacion se reinicia.

## @Column

Personaliza el mapeo entre un campo Java y una columna de la base de datos.

```java
@Column(name = "nombre_completo", nullable = false, length = 100, unique = true)
private String nombre;
```

Atributos mas relevantes:

| Atributo | Descripcion |
|---|---|
| `name` | Nombre de la columna en la base de datos |
| `nullable` | Si la columna acepta NULL |
| `length` | Longitud maxima (solo para `String`) |
| `unique` | Restriccion UNIQUE |
| `precision` | Precision total para tipos numericos |
| `scale` | Escala (decimales) para tipos numericos |
| `insertable` / `updatable` | Controla si el campo participa en INSERT/UPDATE |

> Si no se especifica `@Column`, JPA utiliza el nombre del campo Java como nombre de columna, aplicando la convencion de nomenclatura configurada (`PhysicalNamingStrategy` en Hibernate).

## @Temporal y @Enumerated

`@Temporal` especifica el tipo temporal exacto para campos `java.util.Date` o `java.util.Calendar`. Con `java.time` (Java 8+), JPA mapea automaticamente `LocalDate`, `LocalDateTime` e `Instant` sin necesidad de `@Temporal`.

```java
@Temporal(TemporalType.TIMESTAMP)
private Date fechaCreacion;

// Con Java Time API no hace falta @Temporal
private LocalDateTime ultimoAcceso;
private LocalDate fechaNacimiento;
```

`@Enumerated` controla como se persiste un `enum`. Por defecto usa `ORDINAL` (el indice numerico del enum), lo cual es fragil: si se reordenan o insertan valores en el enum, los datos existentes se corrompen. `STRING` persiste el nombre del valor, que es mas seguro pero ocupa mas espacio.

```java
@Enumerated(EnumType.STRING)
private RolUsuario rol;
```

## Relaciones entre entidades

Las relaciones modelan las asociaciones entre tablas de la base de datos. JPA soporta las cuatro cardinalidades estandar.

### @ManyToOne

Es la relacion mas frecuente. Varias entidades hijas apuntan a una entidad padre. Se mapea con una clave foranea en la tabla hija.

```java
@Entity
public class Pedido {
    @ManyToOne
    @JoinColumn(name = "usuario_id")
    private Usuario usuario;
}
```

`@JoinColumn` indica la columna de clave foranea en la tabla `pedidos`. Si se omite, JPA genera un nombre por defecto (ej. `usuario_id`).

### @OneToMany

Una entidad padre tiene multiples entidades hijas. Siempre se mapea con `mappedBy` para indicar el campo en la entidad hija que posee la clave foranea.

```java
@Entity
public class Usuario {
    @OneToMany(mappedBy = "usuario")
    private List<Pedido> pedidos = new ArrayList<>();
}
```

El `mappedBy` indica que la relacion ya esta mapeada por el campo `usuario` en la entidad `Pedido`. Sin `mappedBy`, JPA espera una tabla intermedia (relacion unidireccional), lo que suele ser incorrecto.

### @OneToOne

Relacion uno a uno. Una entidad tiene exactamente una instancia de otra.

```java
@Entity
public class Usuario {
    @OneToOne
    @JoinColumn(name = "perfil_id")
    private Perfil perfil;
}
```

En el lado inverso, se usa `mappedBy` igual que en `@OneToMany`.

### @ManyToMany

Relacion muchos a muchos. Requiere una tabla intermedia que JPA puede generar automaticamente con `@JoinTable`.

```java
@Entity
public class Usuario {
    @ManyToMany
    @JoinTable(
        name = "usuario_rol",
        joinColumns = @JoinColumn(name = "usuario_id"),
        inverseJoinColumns = @JoinColumn(name = "rol_id")
    )
    private List<Rol> roles = new ArrayList<>();
}
```

`joinColumns` define la columna de la tabla intermedia que referencia a la entidad propietaria, e `inverseJoinColumns` la que referencia a la entidad destino.

> En relaciones `@ManyToMany`, una de las partes debe ser la propietaria (la que define el `@JoinTable`). La otra parte usa `mappedBy` y no define la tabla. La eleccion es arbitraria pero debe ser consistente.

## Cascade

El atributo `cascade` propaga operaciones desde una entidad padre a sus entidades hijas. Sin cascade, cada operacion debe ejecutarse explicitamente sobre cada entidad.

```java
@OneToMany(mappedBy = "usuario", cascade = CascadeType.ALL)
private List<Pedido> pedidos;
```

Tipos de Cascade:

| Tipo | Operacion propagada |
|---|---|
| `PERSIST` | `EntityManager.persist()` |
| `MERGE` | `EntityManager.merge()` |
| `REMOVE` | `EntityManager.remove()` |
| `REFRESH` | `EntityManager.refresh()` |
| `DETACH` | `EntityManager.detach()` |
| `ALL` | Todas las anteriores |

> Usar `CascadeType.ALL` o `CascadeType.REMOVE` en `@ManyToMany` o en relaciones donde la entidad hija tenga otros padres puede eliminar datos que otras entidades necesitan. Cascade no es una conveniencia para ahorrar codigo; es una declaracion de ciclo de vida compartido.

## Fetch types

El `FetchType` controla cuando se cargan los datos de una relacion: inmediatamente al cargar la entidad padre (EAGER) o bajo demanda cuando se accede por primera vez a la relacion (LAZY).

```java
@OneToMany(mappedBy = "usuario", fetch = FetchType.LAZY)
private List<Pedido> pedidos;
```

| FetchType | Comportamiento | Recomendado para |
|---|---|---|
| `LAZY` | Carga la relacion solo cuando se accede a ella | `@OneToMany`, `@ManyToMany` |
| `EAGER` | Carga la relacion junto con la entidad principal | `@ManyToOne`, `@OneToOne` |

LAZY es el valor por defecto para `@OneToMany` y `@ManyToMany`; EAGER lo es para `@ManyToOne` y `@OneToOne`. Sin embargo, confiar en los valores por defecto sin declararlos explicitamente es una mala practica: el comportamiento queda oculto y cualquier cambio futuro en la anotacion altera la semantica de carga.

> El EAGER loading es una de las causas mas frecuentes de problemas de rendimiento en JPA. Cargar una entidad `Pedido` con `@ManyToOne(fetch = EAGER)` sobre `usuario` provoca un JOIN que puede arrastrar a toda la base de datos si las relaciones en cadena tambien son EAGER. LAZY es casi siempre la opcion correcta, salvo en relaciones `@ManyToOne` donde la entidad hija no tiene sentido sin la padre.
