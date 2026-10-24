# Bloqueo y concurrencia

En aplicaciones con multiples usuarios o procesos concurrentes, el acceso simultaneo a los mismos datos puede producir inconsistencias como lecturas sucias, actualizaciones perdidas o lecturas no repetibles. Hibernate proporciona dos mecanismos principales para controlar la concurrencia: el bloqueo optimista (optimistic locking) y el bloqueo pesimista (pessimistic locking).

## Optimistic Locking

El bloqueo optimista asume que los conflictos son poco frecuentes y, en lugar de bloquear los datos en la base de datos, detecta si ha habido modificaciones concurrentes en el momento de la escritura. Si se detecta una modificacion, se lanza una `OptimisticLockException` que la aplicacion debe manejar.

La implementacion mas habitual utiliza el atributo `@Version`, que JPA exige sea de tipo `int`, `Integer`, `short`, `Short`, `long`, `Long`, `java.sql.Timestamp` o `Instant`.

```java
@Entity
@Table(name = "productos")
public class Producto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column
    private String nombre;

    @Column
    private BigDecimal precio;

    @Version
    private Integer version;
}
```

Cuando Hibernate actualiza una entidad con `@Version`, incluye el valor de version en la clausula WHERE del UPDATE:

```sql
UPDATE productos SET nombre = ?, precio = ?, version = 2
WHERE id = ? AND version = 1
```

Si otra transaccion ha modificado la misma fila, el numero de filas afectadas sera 0 (porque el WHERE no coincide) y Hibernate lanzara `OptimisticLockException` o `StaleObjectStateException`.

```java
try {
    producto.setPrecio(new BigDecimal("150.00"));
    session.flush();
} catch (OptimisticLockException e) {
    // Reintentar la operacion: recargar la entidad y aplicar de nuevo los cambios
    session.clear();
    Producto actualizado = session.get(Producto.class, producto.getId());
    actualizado.setPrecio(new BigDecimal("150.00"));
}
```

> La estrategia de reintento debe implementarse con cuidado. No todas las operaciones son idempotentes (por ejemplo, incrementar un contador acumulativo). En esos casos, el reintento simple puede producir resultados incorrectos.

## OptimisticLockException y reintentos

El manejo de `OptimisticLockException` es responsabilidad de la capa de aplicacion. Un patron comun en aplicaciones web es capturar la excepcion en el controlador, recargar la entidad con los valores actuales de la base de datos y presentar de nuevo el formulario al usuario para que resuelva el conflicto manualmente.

En servicios con logica de negocio, se puede implementar un reintento automatico con un numero maximo de intentos:

```java
public void actualizarPrecio(Long productoId, BigDecimal nuevoPrecio) {
    int intentos = 0;
    int maxIntentos = 3;
    while (intentos < maxIntentos) {
        try {
            Producto producto = session.get(Producto.class, productoId);
            producto.setPrecio(nuevoPrecio);
            session.flush();
            return;
        } catch (OptimisticLockException e) {
            intentos++;
            session.clear();
            if (intentos >= maxIntentos) {
                throw new ConcurrenciaExcepcion(
                    "No se pudo actualizar el producto despues de " + maxIntentos + " intentos");
            }
        }
    }
}
```

## Pessimistic Locking

El bloqueo pesimista asume que los conflictos son probables y bloquea directamente las filas en la base de datos, impidiendo que otras transacciones las modifiquen (o incluso las lean, segun el nivel de bloqueo) mientras dure la transaccion.

Hibernate soporta los modos de bloqueo definidos en `LockMode`:

- **PESSIMISTIC_READ**: Adquiere un lock compartido. Otras transacciones pueden leer pero no escribir.
- **PESSIMISTIC_WRITE**: Adquiere un lock exclusivo. Otras transacciones no pueden leer ni escribir (depende del motor de BD).
- **PESSIMISTIC_FORCE_INCREMENT**: Similar a PESSIMISTIC_WRITE pero ademas incrementa la version de la entidad.

### Bloqueo mediante find()

```java
// Bloqueo exclusivo al cargar la entidad
Producto producto = session.find(Producto.class, id,
    LockModeType.PESSIMISTIC_WRITE);
```

### Bloqueo mediante lock()

```java
// Bloqueo sobre una entidad ya cargada
Producto producto = session.get(Producto.class, id);
session.lock(producto, LockModeType.PESSIMISTIC_WRITE);
```

### Bloqueo mediante query.setLockMode()

```java
List<Producto> productos = session.createQuery(
    "FROM Producto p WHERE p.categoria = :cat", Producto.class)
    .setLockMode(LockModeType.PESSIMISTIC_WRITE)
    .setHint("jakarta.persistence.lock.timeout", 5000) // timeout en ms
    .setParameter("cat", "ELECTRONICA")
    .list();
```

> `setHint("jakarta.persistence.lock.timeout", valor)` permite controlar el tiempo maximo que la transaccion esperara para adquirir el lock. Si se supera el timeout, se lanza `LockTimeoutException`. Un valor de 0 significa "esperar infinitamente" y un valor negativo significa "usar el default del motor de BD".

## Deadlock detection

Un deadlock (bloqueo mutuo) ocurre cuando dos transacciones esperan cada una por un recurso que la otra tiene bloqueado. La mayoria de los motores de base de datos detectan deadlocks automaticamente y resuelven el conflicto eligiendo una transaccion como victima, que recibe un error y debe reintentar.

Para minimizar los deadlocks:

- Acceder a las tablas y filas siempre en el mismo orden en todas las transacciones.
- Mantener las transacciones lo mas cortas posible.
- Utilizar bloqueo optimista siempre que sea viable, reservando el bloqueo pesimista para casos donde el coste de un conflicto sea muy alto.
- Establecer timeouts de lock razonables.

```java
// Buen patron: siempre bloquear en el mismo orden
public void transferir(Long cuentaOrigenId, Long cuentaDestinoId, BigDecimal monto) {
    // Ordenar los IDs para evitar deadlocks
    List<Long> ids = List.of(cuentaOrigenId, cuentaDestinoId);
    List<Long> ordenados = ids.stream().sorted().toList();

    Cuenta c1 = session.find(Cuenta.class, ordenados.get(0),
        LockModeType.PESSIMISTIC_WRITE);
    Cuenta c2 = session.find(Cuenta.class, ordenados.get(1),
        LockModeType.PESSIMISTIC_WRITE);

    c1.debitar(monto);
    c2.acreditar(monto);
}
```

## Timeout de bloqueo

El timeout de bloqueo se especifica mediante el hint `jakarta.persistence.lock.timeout` o la propiedad Hibernate `hibernate.jpa.lock.timeout`. El valor se expresa en milisegundos.

```properties
# En application.properties
spring.jpa.properties.hibernate.jpa.lock.timeout=3000
```

```java
// Por entidad
session.find(Producto.class, id, LockModeType.PESSIMISTIC_WRITE,
    Map.of("jakarta.persistence.lock.timeout", 2000));
```

> El comportamiento exacto del timeout depende del motor de base de datos. En PostgreSQL, `lock_timeout` se mide en milisegundos y aplica a nivel de transaccion. En MySQL, depende del motor de almacenamiento (InnoDB lo soporta, MyISAM no).

## Comparacion de estrategias

| Aspecto | Optimistic | Pessimistic |
|---------|------------|-------------|
| Suposicion | Bajos conflictos | Altos conflictos |
| Rendimiento | Alto (sin locks en BD) | Menor (locks en BD) |
| Deteccion | Al hacer flush | Al adquirir el lock |
| Escalabilidad | Alta | Menor |
| Complejidad | Baja (solo @Version) | Media |

La regla general es usar bloqueo optimista como opcion por defecto y reservar el bloqueo pesimista para operaciones criticas donde la consistencia inmediata es obligatoria y el coste de un reintorno es inaceptable, como sistemas financieros o de reservas.
