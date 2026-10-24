# Diseno de entidades

Las entidades JPA son la representacion del modelo de dominio en la base de datos relacional. Un diseno cuidadoso de las entidades evita problemas de rendimiento, integridad y mantenimiento a largo plazo. Cada decision sobre constructores, metodos `equals`/`hashCode`, `toString` y acceso a propiedades tiene implicaciones directas en como Hibernate gestiona el ciclo de vida de los objetos y en como interactuan con los proxies.

## Constructores vacios y completos

JPA exige que toda entidad tenga un constructor vacio (sin argumentos) protegido o publico. Hibernate lo utiliza al crear instancias mediante reflection, especialmente al materializar resultados de consultas. Si no se define explicitamente, el compilador Java genera uno publico automaticamente, pero es una buena practica declararlo explícitamente.

```java
@Entity
@Table(name = "usuarios")
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String nombre;

    @Column(name = "activo")
    private boolean activo;

    protected Usuario() {
        // constructor vacio requerido por JPA
    }

    public Usuario(String email, String nombre) {
        this.email = Objects.requireNonNull(email, "email no puede ser null");
        this.nombre = Objects.requireNonNull(nombre, "nombre no puede ser null");
        this.activo = true;
    }

    // getters y setters...
}
```

> El constructor vacio debe ser `protected` en lugar de `public` para mantener el encapsulamiento, permitiendo que solo JPA (mediante reflection) y las subclases puedan invocarlo. El constructor con parametros fuerza la creacion de instancias validas desde el inicio.

## equals y hashCode basado en ID

Las entidades JPA se comparan en colecciones, en el contexto de persistencia (first-level cache) y en operaciones de merge. Una implementacion incorrecta de `equals`/`hashCode` puede provocar duplicados en `Set`, fallos en la deteccion de cambios o excepciones `LazyInitializationException`. La recomendacion estandar es basar estos metodos exclusivamente en el identificador de la entidad.

```java
@Entity
public class Usuario {

    @Id
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Usuario usuario = (Usuario) o;
        return id != null && Objects.equals(id, usuario.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

- Si `id` es `null` (entidad no persistida aun), `equals` devuelve `false` incluso comparandola consigo misma (salvo que sea la misma referencia). Esto es aceptable porque las entidades transitorias no deberian almacenarse en colecciones que requieran igualdad.
- `hashCode` retorna una constante basada en la clase para que todas las entidades no persistidas tengan el mismo codigo hash y se almacenen en el mismo bucket del `Set`, evitando problemas de insercion cuando se persisten y obtienen un ID.

> No se debe usar campos de negocio (email, nombre) en `equals`/`hashCode`. Si el email cambia, la entidad desapareceria de la coleccion donde esta insertada, y ademas Hibernate puede modificar estos valores durante el flushing, provocando inconsistencias en el hash.

## toString sin relaciones

El metodo `toString` es util para depuracion y logging, pero si incluye relaciones lazy, al invocarlo fuera de una transaccion se lanza `LazyInitializationException`. La solucion es limitar `toString` a los campos simples y excluir explicitamente las colecciones y asociaciones.

```java
@Override
public String toString() {
    return "Usuario{" +
           "id=" + id +
           ", email='" + email + '\'' +
           ", nombre='" + nombre + '\'' +
           ", activo=" + activo +
           '}';
}
```

Si se necesita incluir informacion de relaciones, se puede implementar un metodo especifico (`toStringConDetalles()`) que se invoque dentro de una transaccion activa, o usar DTOs para la representacion externa.

## Getters y setters

Todas las propiedades de una entidad deben tener getters y setters publicos (o protegidos). Hibernate los utiliza para acceder a los valores de las propiedades y para asignarlos al construir entidades desde resultados de consultas. La estrategia de acceso por defecto se determina segun donde se coloque la anotacion `@Id`:

- **Acceso por campo**: `@Id` sobre el atributo. Hibernate accede directamente al campo mediante reflection. No requiere getters/setters, pero es buena practica tenerlos.
- **Acceso por propiedad**: `@Id` sobre el getter. Hibernate usa los getters/setters para acceder y modificar los valores.

La mayoria de proyectos modernos usan acceso por campo por su simplicidad:

```java
@Entity
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
}
```

> Los setters no deben contener logica de negocio ni validaciones que puedan interferir con el funcionamiento interno de Hibernate durante la hidratacion de entidades. La validacion se realiza en los constructores o en metodos de servicio.

## Inmutabilidad

Ciertas entidades no deberian modificarse una vez creadas: eventos historicos, logs de auditoria, transacciones consolidadas. JPA soporta inmutabilidad mediante varias tecnicas:

```java
@Entity
@Table(name = "eventos_auditoria")
@Immutable  // Hibernate especifico: evita UPDATEs y DELETE
public class EventoAuditoria {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, updatable = false)
    private String accion;

    @Column(nullable = false, updatable = false)
    private LocalDateTime timestamp;

    @Column(nullable = false, updatable = false)
    private String usuario;

    public EventoAuditoria(String accion, String usuario) {
        this.accion = accion;
        this.usuario = usuario;
        this.timestamp = LocalDateTime.now();
    }

    protected EventoAuditoria() {}

    public Long getId() { return id; }
    public String getAccion() { return accion; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public String getUsuario() { return usuario; }

    // Sin setters: la entidad es inmutable desde la perspectiva del dominio
}
```

- `@Immutable` de Hibernate evita que el ORM genere sentencias `UPDATE` o `DELETE` para la entidad. Cualquier intento de modificar o eliminar lanza una excepcion.
- Omitir los setters de los campos (excepto `id` si es necesario) refuerza la inmutabilidad a nivel de compilacion.
- Alternativamente, se puede usar `@Column(updatable = false)` en campos individuales para evitar que se incluyan en las sentencias `UPDATE`.

El diseno cuidadoso de entidades no es una preocupacion academica: afecta directamente a la robustez de las consultas, al rendimiento del contexto de persistencia y a la claridad del modelo de dominio. Cada constructor, cada metodo de igualdad y cada getter deben pensarse teniendo en cuenta como Hibernate va a interactuar con ellos.
