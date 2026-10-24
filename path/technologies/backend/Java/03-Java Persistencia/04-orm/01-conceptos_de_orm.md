# Conceptos de ORM

El desajuste de impedancia objeto-relacional es el problema fundamental que toda aplicacion empresarial debe resolver: las bases de datos relacionales almacenan datos en tablas, filas y columnas con tipos primitivos, mientras que el codigo Java trabaja con grafos de objetos interconectados, herencia, polimorfismo y colecciones. Un mapeador objeto-relacional (ORM) actua como puente entre ambos mundos, traduciendo automaticamente las operaciones sobre objetos en sentencias SQL y viceversa, eliminando la necesidad de escribir capas enteras de persistencia manual.

## Que es ORM

ORM (Object-Relational Mapping) es una tecnica de programacion que permite convertir datos entre el sistema de tipos de un lenguaje orientado a objetos y el modelo relacional de una base de datos. En el ecosistema Java, un framework ORM expone una API basada en objetos para que el desarrollador interactue con la base de datos sin escribir SQL explicito. La unidad fundamental de trabajo es la entidad: una clase Java anotada cuyas instancias se corresponden con filas de una tabla y cuyos atributos se corresponden con columnas.

Internamente, el ORM mantiene un contexto de persistencia (session o EntityManager) que rastrea el estado de cada entidad cargada. Cuando se modifica un objeto, el ORM detecta los cambios y genera las sentencias UPDATE necesarias al sincronizar. Esto contrasta con el enfoque tradicional donde cada operacion requiere una sentencia SQL explicita y su correspondiente mapeo manual de resultados.

```java
// Ejemplo de entidad ORM
@Entity
@Table(name = "usuarios")
public class Usuario {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String email;

    @Column(name = "nombre_completo")
    private String nombreCompleto;

    // getters y setters
}
```

## Por que ORM

Eliminar SQL manual de una aplicacion no es un capricho estetico, sino una decision con consecuencias arquitectonicas profundas. El codigo JDBC tradicional fuerza a mezclar logica de negocio con detalles de infraestructura: abrir conexiones, preparar sentencias, iterar sobre ResultSet, cerrar recursos, manejar excepciones SQLException. Cada consulta requiere al menos cinco lineas de codigo boilerplate que nada aportan al dominio del problema.

```java
// JDBC manual: tedioso y propenso a errores
public Usuario buscarPorEmail(String email) {
    String sql = "SELECT id, email, nombre_completo FROM usuarios WHERE email = ?";
    try (Connection conn = dataSource.getConnection();
         PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, email);
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                Usuario u = new Usuario();
                u.setId(rs.getLong("id"));
                u.setEmail(rs.getString("email"));
                u.setNombreCompleto(rs.getString("nombre_completo"));
                return u;
            }
        }
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
    return null;
}
```

Un ORM reduce esto a `entityManager.createQuery("FROM Usuario WHERE email = :email", Usuario.class).setParameter("email", email).getSingleResult()`. La ganancia en productividad es inmediata, pero el beneficio real es la abstraccion: el codigo de negocio queda desacoplado del dialecto SQL concreto, permitiendo cambiar de base de datos sin modificar las consultas.

## Mapeo de clases a tablas

El ORM establece una correspondencia biunivoca entre una clase Java y una tabla de la base de datos. Cada instancia de la clase (entidad) se serializa en una fila de esa tabla. La anotacion `@Entity` marca la clase como candidata a persistencia, y `@Table` permite especificar el nombre real de la tabla si difiere del nombre de la clase.

```java
@Entity
@Table(name = "pedidos", schema = "ventas")
public class Pedido {
    @Id
    private Long id;
    ...
}
```

El nombre de la tabla se deduce por defecto del nombre de la clase, pero es recomendable explicitarlo siempre para evitar sorpresas con las convenciones de mayusculas/minusculas de cada base de datos. El esquema (schema) tambien es configurable mediante `@Table(schema = "ventas")`.

## Mapeo de atributos a columnas

Cada atributo persistente de la entidad se corresponde con una columna de la tabla. La anotacion `@Column` permite personalizar el nombre de la columna, su longitud, si acepta nulos, si es unica, etc. Sin `@Column`, el ORM deduce el nombre de la columna a partir del nombre del atributo siguiendo las convenciones de nomenclatura configuradas.

```java
@Column(name = "fecha_creacion", nullable = false, updatable = false)
private LocalDateTime fechaCreacion;

@Column(length = 500)
private String descripcion;

@Column(unique = true, length = 100)
private String codigoReferencia;
```

El ORM maneja automaticamente la conversion de tipos Java a tipos SQL: `String` a `VARCHAR`, `Integer` a `INTEGER`, `LocalDate` a `DATE`, etc. Cuando el tipo no tiene una correspondencia directa, se recurre a convertidores personalizados mediante `@Convert` o a la anotacion `@Lob` para objetos grandes.

## Relaciones

Las bases de datos relacionales modelan las asociaciones mediante claves ajenas (foreign keys). En Java, las relaciones entre entidades se expresan como referencias a otros objetos o colecciones. El ORM traduce estas referencias a JOINs en SQL, abstractiendo completamente la gestion de claves.

Las cardinalidades se modelan con las anotaciones `@OneToOne`, `@OneToMany`, `@ManyToOne` y `@ManyToMany`. Cada una admite configuracion de carga perezosa (lazy loading), cascada de operaciones y mapeo bidireccional.

```java
@Entity
public class Pedido {
    @Id
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cliente_id")
    private Cliente cliente;

    @OneToMany(mappedBy = "pedido", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<LineaPedido> lineas;
}
```

## ORM vs SQL manual vs hibrido

Ninguna de las dos aproximaciones es intrinsecamente superior; la eleccion depende del contexto. SQL manual ofrece control total sobre las consultas generadas y evita la abstraccion del ORM, lo que resulta critico en operaciones con grandes volumenes de datos donde cada milisegundo cuenta. Sin embargo, el coste de mantenimiento se dispara en proyectos con cientos de tablas y relaciones complejas.

Un ORM destaca en aplicaciones donde predominan las operaciones CRUD y el modelo de dominio es rico en relaciones. La productividad del equipo se multiplica porque el ORM se encarga del 80% de las operaciones de persistencia. El problema es que ese 20% restante (consultas complejas, reportes, batch processing) suele generar SQL ineficiente y requiere intervencion manual.

La mayoria de las aplicaciones del mundo real adoptan un enfoque hibrido: ORM para el CRUD y las transacciones del nucleo del negocio, y SQL nativo (consultas nativas del ORM o directamente JDBC con Spring JDBC Template) para reportes, migraciones de datos y operaciones batch. Esta estrategia combina lo mejor de ambos mundos sin caer en dogmatismos.

> Un error frecuente es intentar que el ORM resuelva problemas para los que no fue disenado. Forzar una consulta de diez JOINs a traves de Criteria API suele producir SQL ineficiente y codigo dificil de leer. En esos casos, una consulta nativa bien escrita es siempre superior.
