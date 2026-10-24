# Transacciones con JPA/Hibernate

JPA y Hibernate abstraen la gestion transaccional de JDBC, proporcionando una API mas limpia y automatica. El `EntityManager` de JPA encapsula la conexion JDBC y la transaccion subyacente, permitiendo al desarrollador centrarse en la logica de negocio sin preocuparse por los detalles de bajo nivel.

## EntityManager y EntityTransaction

El `EntityManager` es la puerta de entrada a las operaciones de persistencia en JPA. Cada operacion dentro de una transaccion se asocia al contexto de persistencia, que gestiona el ciclo de vida de las entidades.

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("miUnidad");
EntityManager em = emf.createEntityManager();

EntityTransaction tx = em.getTransaction();
try {
    tx.begin();

    Cliente cliente = new Cliente("Juan", "juan@email.com");
    em.persist(cliente);                    // Entidad en estado managed

    Pedido pedido = new Pedido(cliente);
    em.persist(pedido);

    tx.commit();                            // Sincroniza y confirma
} catch (Exception e) {
    if (tx.isActive()) {
        tx.rollback();
    }
    throw e;
} finally {
    em.close();                             // Libera el EntityManager
}
```

El ciclo de vida de la transaccion esta vinculado al contexto de persistencia. Cuando se llama a `commit()`, Hibernate ejecuta un flush automatico que sincroniza el estado de todas las entidades gestionadas con la base de datos, en el orden correcto para evitar violaciones de integridad referencial.

> `em.close()` no ejecuta rollback automatico. Si la transaccion no se ha confirmado ni revertido antes de cerrar el EntityManager, los cambios pendientes se pierden pero sin liberar los bloqueos de la base de datos, lo que puede causar problemas de concurrencia.

## @Transactional de Spring

Spring simplifica la gestion transaccional con la anotacion `@Transactional`. Cuando se aplica sobre un metodo o clase, Spring crea automaticamente un proxy que inicia una transaccion antes de la ejecucion del metodo y la confirma o revierte al finalizar.

```java
@Service
@Transactional
public class PedidoService {

    @Autowired
    private PedidoRepository pedidoRepository;

    public Pedido crear(Pedido pedido) {
        // Dentro de una transaccion
        return pedidoRepository.save(pedido);
    }

    @Transactional(readOnly = true)
    public Optional<Pedido> buscarPorId(Long id) {
        return pedidoRepository.findById(id);
    }
}
```

### Atributos de @Transactional

La anotacion admite multiples parametros que configuran el comportamiento transaccional:

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    timeout = 30,
    readOnly = false,
    rollbackFor = IllegalArgumentException.class,
    noRollbackFor = BusinessException.class,
    label = "pedido-transaction"
)
public void procesar(Pedido pedido) {
    // Logica de negocio
}
```

**propagation**: define como se comporta la transaccion cuando el metodo es llamado desde otro contexto transaccional. Los valores se detallan en el tema de propagacion de transacciones.

**isolation**: nivel de aislamiento que se aplica a la transaccion. Si no se especifica, se usa el nivel por defecto del motor de base de datos.

**timeout**: tiempo maximo en segundos para la ejecucion de la transaccion. Si se supera, se lanza una `TransactionTimedOutException` y se revierte la transaccion.

```java
@Transactional(timeout = 10)
public void procesoLargo() {
    // Si tarda mas de 10 segundos, se revierte automaticamente
}
```

**readOnly**: indica que la transaccion solo realiza lecturas. Spring y Hibernate optimizan el comportamiento: no se realiza flush al finalizar, y algunas bases de datos evitan bloqueos de escritura.

```java
@Transactional(readOnly = true)
public List<Producto> listarProductos() {
    return productoRepository.findAll();
}
```

**rollbackFor** y **noRollbackFor**: controlan que excepciones provocan rollback. Por defecto, Spring solo hace rollback en `RuntimeException` y `Error`, pero no en excepciones comprobadas (`Exception`).

```java
@Transactional(rollbackFor = Exception.class)
public void actualizar(Cliente cliente) {
    // Cualquier excepcion provoca rollback
}

@Transactional(noRollbackFor = BusinessWarningException.class)
public void procesarPago(Pago pago) {
    // BusinessWarningException no provoca rollback
}
```

> La regla por defecto de Spring (rollback solo para unchecked exceptions) difiere del estandar EJB, que hace rollback para cualquier excepcion. Es importante conocer esta diferencia para evitar transacciones que se confirmen inesperadamente cuando se lanza una excepcion comprobada.

## TransactionTemplate

`TransactionTemplate` ofrece un enfoque programatico para gestionar transacciones con Spring, manteniendo el manejo de recursos y excepciones que proporciona `@Transactional` pero con control explicito.

```java
@Service
public class PedidoService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public Pedido crearConControl(Pedido pedido, Direccion direccion) {
        return transactionTemplate.execute(status -> {
            // Si es necesario, se puede marcar rollback-only
            if (pedido.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
                status.setRollbackOnly();
                return null;
            }

            pedidoRepository.save(pedido);
            direccionRepository.save(direccion);

            return pedido;
        });
    }

    public void actualizarSinRetorno(Pedido pedido) {
        transactionTemplate.executeWithoutResult(status -> {
            pedidoRepository.actualizarEstado(pedido);
        });
    }
}
```

`TransactionTemplate` es util cuando:
- Se necesitan multiples transacciones dentro del mismo metodo.
- La decision de commit/rollback depende de condiciones complejas.
- Se requiere control fino sobre la configuracion transaccional.

```java
// Multiples transacciones en el mismo metodo
public void procesoLotes(List<Pedido> pedidos) {
    TransactionTemplate tt = new TransactionTemplate(transactionManager);
    tt.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    tt.setTimeout(5);

    for (Pedido pedido : pedidos) {
        try {
            tt.execute(status -> {
                procesarPedido(pedido);
                return null;
            });
        } catch (TransactionTimedOutException e) {
            log.warn("Timeout procesando pedido: {}", pedido.getId());
        }
    }
}
```

## Flush y sincronizacion

Hibernate no sincroniza cada operacion individual con la base de datos. En su lugar, acumula los cambios en el contexto de persistencia y los sincroniza en momentos especificos:

- Antes de ejecutar una consulta que pueda verse afectada por cambios pendientes.
- Cuando se llama explicitamente a `em.flush()`.
- En el commit de la transaccion.

```java
@Transactional
public void actualizarPrecios() {
    List<Producto> productos = productoRepository.findAll();

    for (Producto p : productos) {
        p.setPrecio(p.getPrecio().multiply(new BigDecimal("1.10")));
        // Los cambios se acumulan en el contexto de persistencia
    }

    // En el commit, Hibernate genera un UPDATE por cada producto modificado
}
```

Este comportamiento es eficiente porque Hibernate puede agrupar las sentencias SQL y ejecutarlas en un unico batch, reduciendo el numero de viajes a la base de datos.

> Modificar una entidad fuera de una transaccion provoca una excepcion `LazyInitializationException` al intentar acceder a propiedades perezosas, pero los cambios no se sincronizan con la base de datos porque no hay un contexto de persistencia activo que gestione el flush.
