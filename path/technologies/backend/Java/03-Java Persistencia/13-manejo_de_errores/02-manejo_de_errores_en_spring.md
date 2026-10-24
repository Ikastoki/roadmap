# Manejo de errores en Spring

Spring Data JPA traduce las excepciones propias de cada proveedor ORM (Hibernate, EclipseLink, etc.) a una jerarquia uniforme centrada en `DataAccessException`. Esta capa de traduccion aisla al codigo de aplicacion de las particularidades del motor de persistencia subyacente y permite implementar estrategias de manejo de errores consistentes. Spring proporciona ademas mecanismos declarativos y programaticos para gestionar estos errores en los distintos niveles de la aplicacion.

## @ControllerAdvice y @ExceptionHandler

Para exponer errores de persistencia al cliente HTTP (API REST) se utiliza `@ControllerAdvice`, un componente que intercepta excepciones lanzadas por cualquier controlador y las transforma en respuestas con codigo HTTP y cuerpo adecuados. Combinado con `@ExceptionHandler` se define la logica de traduccion para cada tipo de excepcion.

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> manejarViolacionIntegridad(
            DataIntegrityViolationException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.CONFLICT.value(),
            "La operacion viola una restriccion de la base de datos"
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> manejarEntidadNoEncontrada(
            EntityNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ResponseEntity<ErrorResponse> manejarBloqueoOptimista(
            OptimisticLockingFailureException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.CONFLICT.value(),
            "El recurso ha sido modificado por otro usuario"
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> manejarExcepcionGenerica(Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Error interno del servidor"
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## Manejo de errores en repositories

Cuando se trabaja directamente con `JpaRepository` o `CrudRepository`, las excepciones traducidas por Spring Data empaquetan la causa original. Es importante no capturar `RuntimeException` generico en el repositorio, sino dejar que la excepcion ascienda a la capa de servicio donde se tiene contexto suficiente para decidir si reintentar, compensar o notificar al usuario.

```java
@Repository
public class PedidoRepositoryImpl implements PedidoRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    public void actualizarEstado(Long pedidoId, String estado) {
        try {
            Query query = em.createQuery(
                "UPDATE Pedido p SET p.estado = :estado WHERE p.id = :id"
            );
            query.setParameter("estado", estado);
            query.setParameter("id", pedidoId);
            query.executeUpdate();
        } catch (DataAccessException e) {
            // No se maneja aqui, se relanza o se registra
            throw new PedidoException("No se pudo actualizar el pedido", e);
        }
    }
}
```

## Retry logic para deadlocks

Los deadlocks son inevitables en sistemas con alta concurrencia. La estrategia mas efectiva es detectar la excepcion y reintentar la operacion completa con un backoff exponencial. Spring Retry proporciona `@Retryable` para implementar esta politica de forma declarativa.

```java
@Service
public class PagoService {

    @Retryable(
        retryFor = {CannotAcquireLockException.class, DeadlockLoserDataAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void procesarPago(PagoRequest request) {
        Pago pago = new Pago(request);
        pagoRepository.save(pago);
        cuentaRepository.actualizarSaldo(request.getCuentaId(), request.getImporte());
    }
}
```

> La anotacion `@Retryable` requiere la dependencia `spring-retry` y la habilitacion via `@EnableRetry` en una clase de configuracion. Es fundamental que la operacion sea idempotente para que los reintentos no generen efectos secundarios multiples.

## Circuit breaker

En sistemas distribuidos donde la base de datos puede volverse inaccesible temporalmente, el patron circuit breaker evita que el cliente siga realizando llamadas fallidas saturando el sistema. Con Resilience4j se puede envolver la operacion de persistencia en un circuito que se abre cuando el ratio de fallos supera un umbral.

```java
@CircuitBreaker(name = "database", fallbackMethod = "fallbackGuardar")
public Usuario guardarUsuario(Usuario usuario) {
    return usuarioRepository.save(usuario);
}

public Usuario fallbackGuardar(Usuario usuario, Throwable t) {
    log.error("No se pudo guardar el usuario, encolando para reintento", t);
    colaDeReintentos.encolar(usuario);
    throw new ServicioNoDisponibleException("Base de datos no disponible");
}
```

La configuracion del circuit breaker se define en `application.yml`:

```yaml
resilience4j.circuitbreaker:
  instances:
    database:
      slidingWindowSize: 10
      failureRateThreshold: 50
      waitDurationInOpenState: 10s
      permittedNumberOfCallsInHalfOpenState: 3
```

Combinando `@ControllerAdvice` para la capa de presentacion, `@Retryable` para fallos recuperables y circuit breaker para fallos graves, se obtiene una gestion de errores robusta que mantiene la resiliencia del sistema ante cualquier eventualidad en la persistencia.
