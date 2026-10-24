# Callbacks y Eventos

Los callbacks son metodos que se ejecutan automaticamente en momentos especificos del ciclo de vida de una entidad JPA. Hibernate implementa el soporte completo de callbacks definido por la especificacion JPA y ademas expone su propia API de eventos para escenarios mas avanzados que requieren intervencion en operaciones de bajo nivel.

## Callbacks JPA

Las anotaciones de callback se colocan sobre metodos de la propia entidad o sobre metodos de una clase externa anotada con `@EntityListeners`. El metodo debe ser void, no recibir parametros (si esta definido en la entidad) o recibir la entidad como parametro (si esta en un EntityListener).

```java
@Entity
@Table(name = "usuarios")
@EntityListeners(AuditListener.class)
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column
    private String nombre;

    @Column
    private LocalDateTime fechaCreacion;

    @Column
    private LocalDateTime fechaModificacion;

    @PrePersist
    public void asignarFechaCreacion() {
        this.fechaCreacion = LocalDateTime.now();
        this.fechaModificacion = LocalDateTime.now();
    }

    @PreUpdate
    public void asignarFechaModificacion() {
        this.fechaModificacion = LocalDateTime.now();
    }

    @PostLoad
    public void despuesDeCargar() {
        System.out.println("Usuario cargado desde BD: " + this.nombre);
    }
}
```

### Anotaciones disponibles

- **@PrePersist**: Se ejecuta antes de que Hibernate genere el INSERT. Util para asignar valores por defecto, timestamps de creacion o validar datos antes de persistir.
- **@PostPersist**: Se ejecuta despues de que el INSERT se ha ejecutado. El ID generado ya esta disponible.
- **@PreUpdate**: Se ejecuta antes de que Hibernate genere un UPDATE. Util para actualizar timestamps de modificacion.
- **@PostUpdate**: Se ejecuta despues de ejecutar el UPDATE.
- **@PreRemove**: Se ejecuta antes de generar un DELETE. Puede lanzar una excepcion para cancelar la eliminacion.
- **@PostRemove**: Se ejecuta despues de ejecutar el DELETE.
- **@PostLoad**: Se ejecuta despues de que la entidad se carga desde la base de datos, ya sea por `find()`, `get()`, `load()` o como resultado de una consulta.

> Si un callback lanza una excepcion unchecked (RuntimeException), la transaccion se marca como rollback-only y la operacion no se completa. Esto es util para validaciones que deben impedir la persistencia, pero debe manejarse con cuidado porque cualquier excepcion inesperada en un callback puede romper la transaccion.

### EntityListeners

Cuando la logica de callback debe reutilizarse entre multiples entidades o se prefiere mantenerla separada del modelo, se define en una clase externa anotada con `@EntityListeners`:

```java
public class AuditListener {

    @PrePersist
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setFechaCreacion(LocalDateTime.now());
            auditable.setUsuarioCreacion(AuditContext.getUsuarioActual());
        }
    }

    @PreUpdate
    public void preUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setFechaModificacion(LocalDateTime.now());
            auditable.setUsuarioModificacion(AuditContext.getUsuarioActual());
        }
    }
}
```

> En los metodos de `EntityListeners`, el primer parametro es la entidad. El metodo puede recibir tambien un segundo parametro de tipo `Session` o `EntityManager` si se necesita acceder al contexto de persistencia.

## Interceptor

Hibernate define la interfaz `Interceptor` (y su equivalente `EmptyInterceptor` como implementacion base) que permite interceptar operaciones a nivel de sesion. A diferencia de los callbacks JPA, que se asocian a entidades concretas, el `Interceptor` se registra en la sesion y recibe llamadas para cualquier operacion de cualquier entidad.

```java
public class AuditoriaInterceptor extends EmptyInterceptor {

    @Override
    public boolean onSave(Object entity, Object id, Object[] state,
                          String[] propertyNames, Type[] types) {
        if (entity instanceof Auditable) {
            for (int i = 0; i < propertyNames.length; i++) {
                if ("fechaCreacion".equals(propertyNames[i])) {
                    state[i] = LocalDateTime.now();
                }
            }
        }
        return true;
    }

    @Override
    public void onDelete(Object entity, Object id, Object[] state,
                         String[] propertyNames, Type[] types) {
        System.out.println("Eliminando: " + entity.getClass().getSimpleName()
            + " con id " + id);
    }

    @Override
    public boolean onLoad(Object entity, Object id, Object[] state,
                          String[] propertyNames, Type[] types) {
        // Se ejecuta cada vez que se carga una entidad
        return false;
    }

    @Override
    public void onFlushDirty(Object entity, Object id, Object[] currentState,
                             Object[] previousState, String[] propertyNames,
                             Type[] types) {
        if (entity instanceof Auditable) {
            for (int i = 0; i < propertyNames.length; i++) {
                if ("fechaModificacion".equals(propertyNames[i])) {
                    currentState[i] = LocalDateTime.now();
                }
            }
        }
    }
}
```

El Interceptor se registra al abrir la sesion:

```java
Session session = sessionFactory
    .withOptions()
    .interceptor(new AuditoriaInterceptor())
    .openSession();
```

> Los metodos del Interceptor devuelven `boolean`. En `onSave()`, `onLoad()` y `onFlushDirty()`, devolver `true` indica que se han modificado los valores en el array `state`, mientras que `false` indica que no hubo cambios.

## Event system

Hibernate expone un sistema de eventos mas fino que el Interceptor, basado en la interfaz `EventListener` y el registro de tipos de evento. Cada operacion de Hibernate dispara uno o varios eventos que pueden ser escuchados por implementaciones personalizadas.

Los tipos de evento incluyen `PreInsertEvent`, `PostInsertEvent`, `PreUpdateEvent`, `PostUpdateEvent`, `PreDeleteEvent`, `PostDeleteEvent`, `PreLoadEvent`, `PostLoadEvent`, `FlushEvent`, `DirtyCheckEvent`, entre otros.

```java
public class ValidacionPreInsert implements PreInsertEventListener {

    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        Object entity = event.getEntity();
        if (entity instanceof Producto) {
            Producto p = (Producto) entity;
            if (p.getPrecio() == null || p.getPrecio().compareTo(BigDecimal.ZERO) <= 0) {
                throw new IllegalArgumentException(
                    "El precio del producto debe ser positivo");
            }
        }
        return false;
    }
}
```

La registracion del listener se hace programaticamente:

```java
SessionFactory sessionFactory = new Configuration()
    .configure("hibernate.cfg.xml")
    .buildSessionFactory();

EventRegistry eventRegistry = sessionFactory.getEventEngine().getEventRegistry();
eventRegistry.addDuplicationStrategy(UniqueEventListenerDuplicationStrategy.INSTANCE);

Integrator integrator = integratorName -> serviceRegistry -> {
    EventListenerRegistry registry = serviceRegistry
        .getService(EventListenerRegistry.class);
    registry.appendListeners(EventType.PRE_INSERT, new ValidacionPreInsert());
};
```

> El sistema de eventos es la opcion mas potente y flexible para extender el comportamiento de Hibernate. Sin embargo, su complejidad solo se justifica cuando los callbacks JPA o el Interceptor no cubren el caso de uso. Para la mayoria de los escenarios de auditoria y validacion, los callbacks JPA son suficientes.
