# Relaciones

El verdadero poder de un ORM emerge al modelar las asociaciones entre entidades. Las bases de datos relacionales materializan las relaciones mediante claves ajenas (foreign keys) y tablas intermedias. JPA abstrae estas construcciones y permite trabajar con referencias a objetos y colecciones, traduciendo automaticamente cada operacion a los JOINs y restricciones de integridad correspondientes.

## @OneToOne

La relacion uno a uno asocia dos entidades donde cada instancia de una se corresponde con exactamente una instancia de la otra. Es el caso de un usuario y su perfil, o un pedido y su factura.

```java
@Entity
public class Usuario {
    @Id
    private Long id;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "perfil_id")
    private Perfil perfil;
}

@Entity
public class Perfil {
    @Id
    private Long id;

    private String biografia;

    @OneToOne(mappedBy = "perfil")
    private Usuario usuario;
}
```

La entidad propietaria de la relacion es la que contiene `@JoinColumn`, es decir, la que tiene la clave ajena en su tabla. La entidad inversa usa `mappedBy` para indicar que la relacion ya esta mapeada por el atributo `perfil` de `Usuario`. Sin `mappedBy`, JPA trataria la relacion como dos independientes y crearia dos claves ajenas.

## @OneToMany y @ManyToOne

Es la relacion mas frecuente en aplicaciones empresariales. Un cliente tiene muchos pedidos (`@OneToMany`), y muchos pedidos pertenecen a un cliente (`@ManyToOne`). Siempre se modela como bidireccional para poder navegar en ambos sentidos.

```java
@Entity
public class Cliente {
    @Id
    private Long id;

    @OneToMany(mappedBy = "cliente", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Pedido> pedidos = new ArrayList<>();

    // Metodo helper para mantener ambos lados sincronizados
    public void addPedido(Pedido pedido) {
        pedidos.add(pedido);
        pedido.setCliente(this);
    }
}

@Entity
public class Pedido {
    @Id
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cliente_id", nullable = false)
    private Cliente cliente;
}
```

`mappedBy` en `@OneToMany` indica que el lado propietario es `Pedido.cliente`. La clave ajena `cliente_id` esta en la tabla `pedidos`. Sin `mappedBy`, JPA crearia una tabla intermedia innecesaria.

> El `mappedBy` siempre se coloca en el lado `One` de la relacion. La clave ajena esta siempre en la tabla del lado `Many`.

## @ManyToMany

Una relacion muchos a muchos requiere una tabla intermedia. Es el caso de cursos y alumnos: un curso tiene muchos alumnos y un alumno puede estar en muchos cursos.

```java
@Entity
public class Curso {
    @Id
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "matriculas",
        joinColumns = @JoinColumn(name = "curso_id"),
        inverseJoinColumns = @JoinColumn(name = "alumno_id")
    )
    private List<Alumno> alumnos = new ArrayList<>();
}

@Entity
public class Alumno {
    @Id
    private Long id;

    @ManyToMany(mappedBy = "alumnos")
    private List<Curso> cursos = new ArrayList<>();
}
```

`@JoinTable` define la tabla intermedia `matriculas`. `joinColumns` apunta a la clave ajena que referencia la tabla del propietario (`Curso`), e `inverseJoinColumns` a la clave del lado inverso (`Alumno`). El lado inverso usa `mappedBy` para evitar duplicar la tabla intermedia.

Si la tabla intermedia necesita columnas adicionales (fecha de matricula, nota, estado), entonces ya no es una relacion `@ManyToMany` simple, sino dos relaciones `@OneToMany` hacia una entidad intermedia `Matricula`.

## fetch: LAZY vs EAGER

El atributo `fetch` determina cuando se cargan los datos de la relacion:

- **LAZY** (carga perezosa): la relacion se carga solo cuando se accede a ella por primera vez. Requiere que la entidad este gestionada (asociada a un EntityManager abierto). Es la opcion por defecto para `@OneToMany`, `@ManyToMany` y `@Basic(fetch=LAZY)`. Fuera de la transaccion, acceder a una relacion lazy lanza `LazyInitializationException`.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "categoria_id")
private Categoria categoria;
```

- **EAGER** (carga ansiosa): la relacion se carga junto con la entidad principal, mediante un JOIN o consultas adicionales. Es el valor por defecto para `@ManyToOne` y `@OneToOne`. Aunque simplifica el acceso, cargar relaciones eager innecesarias degrada el rendimiento.

> La regla practica es: usar LAZY por defecto en todas las relaciones y forzar la carga con `JOIN FETCH` en las consultas JPQL donde se necesiten. El eager loading deberia reservarse para relaciones donde la entidad carece de sentido sin su asociacion (como una `LineaPedido` sin su `Pedido`).

## cascade: propagacion de operaciones

El atributo `cascade` define que operaciones sobre la entidad padre se propagan a las entidades hijas. Los valores del enum `CascadeType` son:

| Tipo | Operacion |
|---|---|
| `PERSIST` | Al persistir el padre, persiste automaticamente los hijos |
| `MERGE` | Al mergear el padre, mergea los hijos |
| `REMOVE` | Al eliminar el padre, elimina los hijos |
| `REFRESH` | Al refrescar el padre, refresca los hijos |
| `DETACH` | Al detached el padre, detached los hijos |
| `ALL` | Todas las anteriores |

```java
@OneToMany(mappedBy = "pedido", cascade = CascadeType.ALL, orphanRemoval = true)
private List<LineaPedido> lineas = new ArrayList<>();
```

`CascadeType.ALL` en una relacion `@OneToMany` hace que persistir un `Pedido` persista automaticamente todas sus `LineaPedido`. Sin cascade, seria necesario persistir cada linea individualmente.

> Usar cascade con cuidado. `CascadeType.REMOVE` (incluido en ALL) en una `@ManyToOne` puede eliminar el padre al eliminar el hijo, lo que rara vez es el comportamiento deseado.

## mappedBy

`mappedBy` indica que la relacion es la contraparte inversa (no propietaria) de una relacion bidireccional. Su valor es el nombre del atributo en la entidad propietaria que referencia a la entidad actual.

```java
// Entidad propietaria (tiene la FK)
public class Pedido {
    @ManyToOne
    @JoinColumn(name = "cliente_id")
    private Cliente cliente;
}

// Entidad inversa (usa mappedBy)
public class Cliente {
    @OneToMany(mappedBy = "cliente")  // "cliente" es el atributo en Pedido
    private List<Pedido> pedidos;
}
```

Si se omite `mappedBy` en el lado `One`, JPA trata la relacion como dos unidireccionales independientes y crea una tabla intermedia adicional. Es uno de los errores de mapeo mas comunes.

## orphanRemoval

`orphanRemoval = true` elimina automaticamente las entidades hijas que quedan huerfanas al ser removidas de la coleccion del padre. Es decir, si se elimina una linea de la lista `pedido.getLineas()`, al sincronizar la transaccion JPA ejecutara un DELETE contra la fila de la linea huerfana.

```java
@OneToMany(mappedBy = "pedido", cascade = CascadeType.ALL, orphanRemoval = true)
private List<LineaPedido> lineas;

// En codigo de negocio
pedido.getLineas().remove(0);  // Al hacer flush(), se elimina de BD
```

Sin `orphanRemoval = true`, la linea se desasocia del pedido pero permanece en la base de datos con `pedido_id = null` (si la columna acepta nulos). `orphanRemoval` solo tiene sentido en relaciones `@OneToMany` con cascade.
