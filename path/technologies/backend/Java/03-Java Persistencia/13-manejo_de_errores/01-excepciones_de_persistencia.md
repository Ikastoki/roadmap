# Excepciones de persistencia

Toda aplicacion que interactua con una base de datos debe contemplar que las operaciones de persistencia pueden fallar por multiples razones: restricciones de integridad, bloqueos concurrentes, timeouts de conexion o fallos transitorios del servidor. JPA y Spring Data definen una jerarquia de excepciones que permiten modelar cada uno de estos escenarios de forma granular, facilitando que el desarrollador reaccione adecuadamente segun la naturaleza del error sin recurrir a bloques catch genericos.

## PersistenceException

Es la excepcion base de toda la jerarquia de JPA. Pertenece a `javax.persistence.PersistenceException` (o `jakarta.persistence.PersistenceException` en Jakarta EE) y extiende `RuntimeException`. Esto implica que no es obligatorio capturarla, aunque si no se maneja la transaccion quedara marcada como `rollback-only` y el EntityManager se volvera inutilizable. PersistenceException encapsula causas subyacentes como `SQLException`, `ConstraintViolationException` o `OptimisticLockException`.

```java
try {
    entityManager.persist(usuario);
    entityManager.flush();
} catch (PersistenceException e) {
    Throwable cause = e.getCause();
    if (cause instanceof SQLException) {
        String sqlState = ((SQLException) cause).getSQLState();
        // 23505 es violacion de unicidad en PostgreSQL
        if ("23505".equals(sqlState)) {
            // manejar duplicado
        }
    }
}
```

## EntityNotFoundException

Se lanza cuando se intenta recuperar una entidad mediante `find` o `getReference` y esta no existe. Con `find` la excepcion solo se produce si se usa `load()` de Hibernate o `getReference()` de JPA, ya que `find` sencillamente devuelve `null`. `getReference()` devuelve un proxy y difiere el acceso a la base de datos, lanzando EntityNotFoundException solo cuando se accede a una propiedad del proxy y el registro no existe.

```java
// No lanza excepcion, devuelve null
Usuario u = em.find(Usuario.class, 999L);

// Lanza EntityNotFoundException al acceder al proxy
Usuario u = em.getReference(Usuario.class, 999L);
String nombre = u.getNombre(); // aqui salta
```

## OptimisticLockException

Aparece cuando dos transacciones intentan actualizar la misma entidad concurrentemente y se detecta un conflicto mediante la version de la entidad. JPA soporta bloqueo optimista anotando un campo con `@Version` (tipicamente `int`, `Integer`, `long`, `Long` o `Timestamp`). Hibernate incrementa automaticamente la version en cada actualizacion y verifica en el `UPDATE` que la version coincida con la original. Si ninguna fila resulta afectada, lanza `OptimisticLockException` o `StaleObjectStateException`.

```java
@Entity
public class Cuenta {
    @Id private Long id;
    private BigDecimal saldo;
    @Version private int version;
}

// en el servicio
@Transactional
public void transferir(Long origenId, Long destinoId, BigDecimal monto) {
    try {
        Cuenta origen = cuentaRepo.findById(origenId).orElseThrow();
        origen.setSaldo(origen.getSaldo().subtract(monto));
        cuentaRepo.save(origen);
    } catch (OptimisticLockException e) {
        // Reintentar la operacion completa
        throw new RetryableException("Conflicto de concurrencia, reintente", e);
    }
}
```

## PessimisticLockException

Se produce cuando se utiliza bloqueo pesimista (mediante `LockModeType.PESSIMISTIC_WRITE` o `PESSIMISTIC_READ`) y el sistema de base de datos no puede adquirir el bloqueo, bien porque hay un deadlock o porque se supera el timeout definido. A diferencia del bloqueo optimista, el pesimista asegura el recurso en el momento de la consulta mediante `SELECT ... FOR UPDATE`, evitando que otras transacciones modifiquen las filas bloqueadas hasta que la transaccion actual finalice.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT c FROM Cuenta c WHERE c.id = :id")
Optional<Cuenta> findByIdConBloqueo(@Param("id") Long id);
```

> El bloqueo pesimista debe usarse con moderacion porque puede degradar la concurrencia y provocar deadlocks. Es preferible para operaciones de corta duracion donde el conflicto es frecuente y la reintentativa optimista resultaria mas costosa.

## TransactionRequiredException

Se lanza cuando se intenta realizar una operacion que requiere una transaccion activa (persist, merge, remove, flush) y no existe ninguna. Ocurre tipicamente al invocar metodos del EntityManager fuera del ambito de `@Transactional` o cuando la transaccion ya ha sido marcada para rollback.

```java
// esto lanza TransactionRequiredException
usuarioService.guardar(usuario); // sin @Transactional

// solucion: anotar el metodo en la capa de servicio
@Transactional
public void guardar(Usuario usuario) {
    entityManager.persist(usuario);
}
```

## RollbackException

Indica que la transaccion ha sido forzada a deshacerse, bien por una excepcion previa o porque se ha llamado a `setRollbackOnly()`. Es comun verla como excepcion envolvente cuando se produce un error durante el `commit()`. Spring Data JPA suele traducir estas excepciones a `DataAccessException` de su propia jerarquia para abstraer al desarrollador del proveedor JPA subyacente.

```java
@Transactional
public void procesar() {
    // si ocurre cualquier RuntimeException, la transaccion
    // se marca como rollback-only y el commit lanza RollbackException
}
```

> Conocer la semantica de cada excepcion permite implementar politicas de reintento selectivas: las `OptimisticLockException` se benefician de reintentos con backoff, mientras que las `EntityNotFoundException` indican un error logico que no se soluciona reintentando.
