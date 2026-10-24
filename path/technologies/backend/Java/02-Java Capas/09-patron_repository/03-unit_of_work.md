# Unit of Work

Una operacion de negocio rara vez se limita a una sola insercion o una sola actualizacion. Crear un pedido implica guardar el pedido mismo, actualizar el stock de los productos, registrar el movimiento contable y, posiblemente, enviar una notificacion. Si cualquiera de estos pasos falla despues de que los anteriores ya se hayan persistido, el sistema queda en un estado inconsistente. El patron Unit of Work resuelve este problema agrupando todas las operaciones de persistencia en una unidad atomica: todas se completan con exito o ninguna deja rastro en la base de datos. Es el mecanismo que materializa el principio ACID en la capa de aplicacion.

## Concepto y responsabilidades

El Unit of Work actua como un registro de todas las operaciones de lectura y escritura realizadas durante una transaccion. Mantiene un seguimiento de las entidades que se han creado, modificado o eliminado, y al final del ciclo de trabajo sincroniza esos cambios con la base de datos en una unica operacion atomica.

El ciclo tipico consta de tres fases. Durante la fase de operaciones, el codigo de negocio invoca metodos del repositorio (`save`, `delete`) que no impactan directamente en la base de datos, sino que marcan las entidades como "pendientes" dentro del Unit of Work. En la fase de `flush`, los cambios pendientes se envian a la base de datos como sentencias SQL, pero sin confirmar la transaccion subyacente. Finalmente, en la fase de `commit`, la transaccion se confirma y los cambios se hacen permanentes. Si ocurre un error en cualquier punto, un `rollback` deshace todas las modificaciones, devolviendo la base de datos al estado anterior al inicio del Unit of Work.

```java
// Pseudocodigo del ciclo
UnitOfWork uow = new UnitOfWork(entityManager);

User user = new User("Juan", "juan@example.com");
Order order = new Order(user);

uow.registerNew(user);
uow.registerNew(order);
uow.registerDirty(order); // Order modificada

try {
    uow.flush();    // Envia INSERT/UPDATE a la BD
    uow.commit();   // Confirma la transaccion
} catch (Exception e) {
    uow.rollback(); // Deshace todo
}
```

> El `flush` no equivale al `commit`. Flush sincroniza el estado del contexto de persistencia con la base de datos, pero los cambios siguen siendo parte de una transaccion abierta y pueden deshacerse con rollback. Commit hace que los cambios sean permanentes y visibles para otras transacciones.

## Flush, commit y rollback en JPA

En JPA, `EntityManager` es la implementacion concreta del Unit of Work. El contexto de persistencia que gestiona actua como un cache transaccional donde las entidades se cargan, modifican y eliminan antes de sincronizarse con la base de datos.

### Flush

El metodo `flush()` fuerza la sincronizacion del contexto de persistencia con la base de datos. Antes del flush, las operaciones como `persist`, `merge` o `remove` solo afectan al contexto en memoria. Despues del flush, las sentencias SQL correspondientes se han ejecutado en la base de datos, pero la transaccion sigue abierta.

```java
EntityManager em = entityManagerFactory.createEntityManager();
em.getTransaction().begin();

User user = new User("Juan", "juan@example.com");
em.persist(user); // Solo en contexto, aun no hay INSERT

em.flush(); // Ahora se ejecuta: INSERT INTO users ...

// La transaccion sigue abierta, podemos hacer rollback si es necesario
```

El flush se activa implicitamente en tres situaciones:

- Antes de cualquier consulta que pueda verse afectada por cambios pendientes (para evitar resultados obsoletos).
- Cuando la transaccion se confirma (`commit`).
- Cuando el programador lo invoca explicitamente mediante `em.flush()`.

La configuracion de `FlushModeType` controla este comportamiento. `FlushModeType.AUTO` (el valor por defecto) ejecuta flush antes de cada consulta. `FlushModeType.COMMIT` solo ejecuta flush en el commit, lo que mejora el rendimiento pero puede devolver resultados desactualizados si se realizan consultas intermedias.

### Commit

`commit()` cierra la transaccion y hace permanentes todos los cambios realizados desde el `begin`. Internamente, JPA ejecuta un flush implicito antes del commit si no se ha realizado ya:

```java
em.getTransaction().begin();
// ... operaciones
em.getTransaction().commit(); // flush implicito + commit JDBC
```

Una vez realizado el commit, las entidades del contexto de persistencia quedan en estado "detached": siguen siendo objetos Java validos, pero ya no estan asociadas al EntityManager. Cualquier modificacion posterior sobre ellas no se reflejara en la base de datos a menos que se fusionen nuevamente con un nuevo contexto.

### Rollback

`rollback()` deshace todos los cambios realizados en la transaccion actual. La base de datos vuelve al estado exacto en el que se encontraba antes del `begin`:

```java
try {
    em.getTransaction().begin();
    em.persist(user1);
    em.persist(user2);
    // Si ocurre un error antes del commit
    em.getTransaction().rollback(); // Ningun usuario se persiste
} catch (Exception e) {
    if (em.getTransaction().isActive()) {
        em.getTransaction().rollback();
    }
}
```

Despues de un rollback, el contexto de persistencia se invalida. Todas las entidades que contenian deben descartarse y, si se necesita continuar operando, debe crearse un nuevo contexto.

## @Transactional en Spring

Spring abstrae la gestion del Unit of Work mediante la anotacion `@Transactional`. Cuando se aplica a un metodo de servicio, Spring intercepta la llamada, abre una transaccion antes de ejecutar el metodo y la confirma o deshace al finalizar, segun el resultado:

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final InventoryService inventoryService;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request.userId());
        orderRepository.save(order);

        for (ItemRequest item : request.items()) {
            Product product = productRepository.findById(item.productId())
                .orElseThrow(() -> new ProductNotFoundException(item.productId()));

            inventoryService.decreaseStock(product, item.quantity());
            order.addItem(product, item.quantity());
        }

        return order;
    }
}
```

Si el metodo completa su ejecucion sin lanzar excepcion, Spring ejecuta `commit`. Si lanza una `RuntimeException` (o cualquier subclase), Spring ejecuta `rollback`. Las excepciones checked no provocan rollback por defecto, aunque puede configurarse con `@Transactional(rollbackFor = Exception.class)`.

> La anotacion `@Transactional` solo tiene efecto cuando se invoca a traves del proxy de Spring. Una llamada interna directa entre metodos de la misma clase (autollamada) no activa la transaccion porque no atraviesa el proxy.

### Propagacion

La propagacion determina como se comporta un metodo transaccional cuando es invocado desde otro metodo que ya tiene una transaccion activa. Spring define siete niveles de propagacion en la enumeracion `Propagation`:

| Propagacion | Comportamiento |
|---|---|
| `REQUIRED` | Usa la transaccion actual; si no existe, crea una nueva. Es el valor por defecto |
| `REQUIRES_NEW` | Suspende la transaccion actual y crea una nueva. La transaccion suspendida se reanuda al finalizar la nueva |
| `SUPPORTS` | Usa la transaccion actual si existe; si no, ejecuta sin transaccion |
| `NOT_SUPPORTED` | Suspende la transaccion actual y ejecuta sin transaccion |
| `MANDATORY` | Usa la transaccion actual; lanza excepcion si no existe |
| `NEVER` | Ejecuta sin transaccion; lanza excepcion si existe una transaccion activa |
| `NESTED` | Crea un punto de salvacion dentro de la transaccion actual. Si la transaccion anidada falla, solo se deshace hasta el punto de salvacion |

`REQUIRES_NEW` resulta util cuando una operacion secundaria debe persistir independientemente de que la operacion principal falle. Por ejemplo, registrar un log de auditoria aunque el proceso principal falle:

```java
@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String action, String detail) {
        // Se persiste incluso si la transaccion padre hace rollback
        auditRepository.save(new AuditLog(action, detail));
    }
}
```

`NESTED` difiere de `REQUIRES_NEW` en que no crea una transaccion independiente, sino un savepoint dentro de la misma transaccion. Si la operacion anidada falla, solo se deshacen sus cambios, pero la transaccion padre puede continuar. El soporte de `NESTED` depende del gestor de base de datos subyacente; no todos los drivers JDBC lo implementan.

### Configuracion adicional

La anotacion `@Transactional` admite parametros adicionales para controlar el comportamiento de la transaccion:

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.REPEATABLE_READ,
    timeout = 30,
    readOnly = true,
    rollbackFor = { DataAccessException.class, BusinessException.class },
    noRollbackFor = { AuditException.class }
)
public List<User> findAllUsers() {
    return userRepository.findAll();
}
```

- `isolation`: define el nivel de aislamiento de la transaccion (`READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`). Controla como los cambios de otras transacciones afectan a la actual.
- `readOnly`: indica que la transaccion solo realizara lecturas. Los gestores de bases de datos pueden optimizar la ejecucion y JPA puede desactivar el dirty checking.
- `timeout`: tiempo maximo en segundos antes de que la transaccion expire y se deshaga automaticamente.
- `rollbackFor` / `noRollbackFor`: personaliza que excepciones provocan rollback.

> Marcar un metodo como `@Transactional(readOnly = true)` no impide que se realicen escrituras, pero permite que el gestor de persistencia optimice el acceso y, en el caso de JPA, evite la carga innecesaria del contexto.
