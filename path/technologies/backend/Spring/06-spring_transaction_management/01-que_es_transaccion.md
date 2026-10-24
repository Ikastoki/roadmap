# Que es transaccion en Spring

Una transaccion en el contexto de bases de datos es una unidad logica de trabajo que agrupa una o mas operaciones de lectura o escritura de tal forma que se comportan atomicamente: o todas se completan con exito, o ninguna tiene efecto. Spring Framework proporciona un modelo de gestion de transacciones que abstrae los mecanismos de transaccion del proveedor de persistencia subyacente, permitiendo al desarrollador trabajar con anotaciones o configuracion declarativa sin preocuparse por el inicio, commit o rollback manual de transacciones.

La gestion de transacciones es uno de los pilares fundamentales de cualquier aplicacion que interactue con fuentes de datos. Sin transacciones, una serie de operaciones sobre la base de datos queda vulnerable a fallos parciales: si se inserta un registro en una tabla y la operacion siguiente sobre otra tabla falla, el sistema queda en un estado inconsistente. Las transacciones garantizan la integridad de los datos al agrupar operaciones relacionadas en una unidad indivisible.

## Propiedades ACID

Las transacciones de bases de datos se rigen por las propiedades ACID, un acronimo que describe las cuatro garantias fundamentales:

**Atomicidad (Atomicity):** Todas las operaciones dentro de una transaccion se ejecutan como una unidad indivisible. Si cualquiera de las operaciones falla, todas las operaciones anteriores se deshacen (rollback). No existe un estado intermedio visible para otros procesos. Este comportamiento es el que garantiza que un transferencia bancaria no pueda debitar una cuenta sin acreditar la otra.

**Consistencia (Consistency):** La transaccion lleva la base de datos de un estado valido a otro estado valido, preservando todas las restricciones de integridad (claves primarias, foraneas, unicos, checks). Si una operacion violaria una restriccion, la transaccion completa se revierte.

**Aislamiento (Isolation):** Las transacciones concurrentes no interfieren entre si. Cada transaccion opera como si fuera la unica sobre la base de datos. El nivel de aislamiento determina hasta que punto las modificaciones de una transaccion son visibles para otras transacciones en curso, y se configura mediante los isolation levels.

**Durabilidad (Durability):** Una vez que una transaccion se ha completado con exito (commit), los cambios son permanentes incluso en caso de fallo del sistema. Esto se garantiza mediante mecanismos de escritura en disco (WAL, journaling) del motor de base de datos.

## Local vs Global Transactions

Las transacciones locales (o de recurso individual) abarcan una unica fuente de datos. Se inician, confirman o revierten sobre una sola conexion a la base de datos. Son el caso de uso mas comun en aplicaciones que trabajan con una unica base de datos y no requieren coordinacion con otros recursos transaccionales como colas de mensajes o sistemas de archivos.

Las transacciones globales (o distribuidas) coordinan multiples recursos transaccionales en una unica transaccion. Utilizan el protocolo X/Open XA o JTA (Java Transaction API) para coordinar el commit o rollback entre multiples bases de datos, colas JMS, y otros recursos. Son significativamente mas complejas en terminos de configuracion y rendimiento, y se reservan para escenarios donde la consistencia atomica entre multiples sistemas es un requisito critico.

```java
// Transaccion local: opera sobre una unica base de datos
@Service
public class PedidoService {

    @Transactional
    public Pedido crearPedido(Pedido pedido) {
        // Ambas operaciones sobre la misma DB
        pedidoRepository.save(pedido);
        inventarioRepository.descontar(pedido.getProductoId(), pedido.getCantidad());
        return pedido;
    }
}
```

> Las transacciones globales son raramente necesarias en aplicaciones modernas. En su lugar, se prefiere el patron Saga o la compensacion de transacciones, donde cada paso tiene una operacion de deshacer en caso de fallo, coordinada a nivel de aplicacion.

## PlatformTransactionManager

`PlatformTransactionManager` es la interfaz central del modelo de transacciones de Spring. Define tres metodos fundamentales: `getTransaction()` para iniciar una transaccion, `commit()` para confirmar los cambios y `rollback()` para deshacerlos. Cada proveedor de persistencia tiene su implementacion de esta interfaz.

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition) 
        throws TransactionException;
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}
```

Spring Boot autoconfigura el `PlatformTransactionManager` adecuado segun las dependencias del classpath. Si detecta `spring-boot-starter-data-jpa`, configura un `JpaTransactionManager`. Si detecta solo JDBC con un DataSource, configura un `DataSourceTransactionManager`. Si se usa R2DBC oSpring Data Reactive, no se usa esta interfaz sino el modelo reactivo de transacciones.

```java
// Implementaciones comunes
DataSourceTransactionManager    // Para JDBC puro o Spring JDBC
JpaTransactionManager           // Para JPA / Hibernate
HibernateTransactionManager     // Para Hibernate directo (hereda de JPA)
```

## TransactionDefinition

`TransactionDefinition` encapsula los parametros que definen el comportamiento de una transaccion: propagacion, aislamiento, timeout, si es de solo lectura y un nombre para identificacion. Cuando se usa `@Transactional`, Spring construye un `TransactionDefinition` a partir de los atributos de la anotacion y lo pasa a `PlatformTransactionManager.getTransaction()`.

```java
// Equivalente programatico de lo que @Transactional hace internamente
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
def.setIsolationLevel(TransactionDefinition.ISOLATION_DEFAULT);
def.setTimeout(30);
def.setReadOnly(false);

TransactionStatus status = transactionManager.getTransaction(def);
try {
    // operaciones de negocio
    transactionManager.commit(status);
} catch (Exception e) {
    transactionManager.rollback(status);
    throw e;
}
```

## TransactionStatus

`TransactionStatus` representa el estado de una transaccion en curso. Proporciona metodos para verificar si la transaccion es nueva (`isNewTransaction()`), si esta marcada como rollback-only (`isRollbackOnly()`) y si tiene savepoints. Tambien permite invocar `setRollbackOnly()` para marcar la transaccion para rollback sin lanzar una excepcion.

```java
TransactionStatus status = transactionManager.getTransaction(def);
try {
    // Si algo falla parcialmente pero no se quiere lanzar excepcion
    if (condicionDeFallo) {
        status.setRollbackOnly(); // marca para rollback al hacer commit
    }
    transactionManager.commit(status); // hara rollback automaticamente
} catch (Exception e) {
    transactionManager.rollback(status);
}
```

## Flujo de una Transaccion Declarativa

El siguiente diagrama describe el flujo que Spring sigue internamente cuando se invoca un metodo anotado con `@Transactional`:

```
Caller invoca metodo @Transactional
    │
    ▼
Spring AOP intercepta la llamada (proxy)
    │
    ▼
¿Existe una transaccion en curso?
    │
    ├── SI → ¿La propagacion permite reusar?
    │         ├── SI → Reusa la transaccion existente
    │         └── NO → Crea nueva transaccion (REQUIRES_NEW, etc.)
    │
    └── NO → Crea nueva transaccion
              │
              ▼
        Ejecuta el metodo de negocio
              │
              ├── Exito → commit()
              │              │
              │              ▼
              │         Transaccion confirmada, cambios persistidos
              │
              └── Excepcion → ¿Es rollbackable?
                              ├── SI → rollback()
                              └── NO → commit() (excepcion propagada pero
                                            transaccion ya commiteada)
```

> Cuando un metodo con `@Transactional` lanza una excepcion, Spring verifica si la excepcion es de tipo `RuntimeException` o `Error` (o si esta en la lista de `rollbackFor`). Solo en ese caso realiza rollback. Las checked exceptions por defecto no provocan rollback, lo cual es una fuente comun de bugs si no se configura correctamente `rollbackFor`.

