# Transacciones en Java

Una transaccion es una unidad logica de trabajo compuesta por una o varias operaciones sobre la base de datos que se ejecutan como un todo indivisible. El sistema de gestion de bases de datos garantiza que, ante cualquier fallo, la transaccion se complete totalmente o no se aplique en absoluto, manteniendo la consistencia de los datos.

Java proporciona multiples mecanismos para trabajar con transacciones, desde el control explicito mediante JDBC hasta la gestion declarativa con Spring. La eleccion del mecanismo adecuado depende de la complejidad de la aplicacion, del modelo de concurrencia y de los requisitos de escalabilidad.

## ACID en Java

Las transacciones se rigen por las propiedades ACID, que en el contexto de Java se implementan de la siguiente forma:

- **Atomicidad**: todas las operaciones dentro de una transaccion se completan o ninguna se aplica. En JDBC se controla con `commit()` y `rollback()`. En JPA, con `EntityTransaction`.
- **Consistencia**: la transaccion lleva la base de datos de un estado valido a otro estado valido, respetando todas las restricciones. Es responsabilidad de la aplicacion asegurar que las reglas de negocio se cumplen dentro de la transaccion.
- **Aislamiento**: las transacciones concurrentes no interfieren entre si. Se controla mediante los niveles de aislamiento definidos en `java.sql.Connection`.
- **Persistencia**: una vez confirmada, la transaccion sobrevive a fallos del sistema. La base de datos garantiza que los datos confirmados se escriben en almacenamiento persistente.

```java
// Representacion de ACID en una operacion bancaria
public void transferir(String origen, String destino, BigDecimal monto) {
    Connection conn = null;
    try {
        conn = dataSource.getConnection();
        conn.setAutoCommit(false);                     // Atomicidad

        // Las reglas de negocio garantizan la consistencia
        cuentaDao.restar(origen, monto);
        cuentaDao.sumar(destino, monto);

        conn.commit();                                 // Persistencia
    } catch (Exception e) {
        if (conn != null) conn.rollback();             // Atomicidad
        throw e;
    }
}
```

## Transacciones locales vs globales

### Transacciones locales

Una transaccion local opera sobre un unico recurso, generalmente una sola base de datos. Se gestiona directamente a traves de la conexion JDBC o del `EntityManager` de JPA.

```java
// Transaccion local con JPA
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();
try {
    em.persist(entidad);
    em.getTransaction().commit();
} catch (Exception e) {
    em.getTransaction().rollback();
    throw e;
} finally {
    em.close();
}
```

Las transacciones locales son simples y eficientes. Son adecuadas para la mayoria de aplicaciones monoliticas que acceden a una sola base de datos.

### Transacciones globales (JTA)

Una transaccion global, gestionada mediante Java Transaction API (JTA), puede abarcar multiples recursos: varias bases de datos, colas de mensajes (JMS) o servicios transaccionales.

```java
// Transaccion global con JTA (API directa)
@Resource
private UserTransaction userTransaction;

public void operacionDistribuida() {
    userTransaction.begin();
    try {
        baseDatosA.insertar(datos);
        baseDatosB.actualizar(datos);
        colaMensajes.enviar(mensaje);
        userTransaction.commit();
    } catch (Exception e) {
        userTransaction.rollback();
        throw e;
    }
}
```

Las transacciones globales utilizan el protocolo two-phase commit (2PC) para garantizar la atomicidad entre recursos. Tienen un coste de rendimiento significativo y solo deben usarse cuando es estrictamente necesario.

## Programmatic vs declarative

### Transacciones programaticas

El control de transacciones se realiza explicitamente en el codigo, invocando `begin()`, `commit()` y `rollback()`. Ofrece control total sobre los limites transaccionales, pero el codigo transaccional se mezcla con la logica de negocio.

```java
// Enfoque programatico con TransactionTemplate
@Service
public class PedidoService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public Pedido crear(Pedido pedido) {
        return transactionTemplate.execute(status -> {
            // Logica de negocio dentro de la transaccion
            pedidoRepository.save(pedido);
            inventarioService.actualizarStock(pedido);
            return pedido;
        });
    }
}
```

### Transacciones declarativas

Mediante anotaciones o configuracion XML, se declaran los limites transaccionales sin modificar la logica de negocio. Spring `@Transactional` es el ejemplo mas representativo.

```java
// Enfoque declarativo con @Transactional
@Service
public class PedidoService {

    @Transactional
    public Pedido crear(Pedido pedido) {
        pedidoRepository.save(pedido);
        inventarioService.actualizarStock(pedido);
        return pedido;
    }
}
```

El enfoque declarativo reduce el codigo boilerplate y separa la preocupacion transaccional de la logica de negocio. Spring utiliza AOP (Aspect Oriented Programming) para envolver el metodo en una transaccion antes de su ejecucion.

> Las transacciones declarativas con `@Transactional` solo funcionan en metodos publicos llamados desde otro bean de Spring. Una llamada interna dentro de la misma clase (auto-invocacion) no activa la logica transaccional porque el proxy AOP no intercepta llamadas internas.
