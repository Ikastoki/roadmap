# Transacciones - buenas practicas

Las transacciones son el mecanismo que garantiza la atomicidad, consistencia, aislamiento y durabilidad (ACID) de las operaciones sobre la base de datos. En aplicaciones Spring con JPA, `@Transactional` es la herramienta declarativa que define los limites transaccionales, pero su colocacion y configuracion incorrectas provocan problemas de rendimiento, integridad y mantenibilidad. La regla fundamental es que la capa de servicio es la unica responsable de gestionar las transacciones.

## @Transactional en capa de servicio

Los servicios son el lugar natural para definir los limites transaccionales porque encapsulan unidades de trabajo que agrupan multiples operaciones sobre repositorios. Colocar `@Transactional` en los repositorios o en los controladores rompe la separacion de responsabilidades y dificulta la composicion de operaciones.

```java
@Service
@Transactional(readOnly = true)
public class PedidoService {

    private final PedidoRepository pedidoRepository;
    private final InventarioRepository inventarioRepository;
    private final EmailService emailService;

    public Pedido buscarPorId(Long id) {
        return pedidoRepository.findById(id)
            .orElseThrow(() -> new PedidoNoEncontradoException(id));
    }

    @Transactional
    public Pedido crearPedido(CrearPedidoRequest request) {
        Pedido pedido = new Pedido(request);
        pedido.asignarCliente(request.getClienteId());

        // Se ejecuta dentro de la misma transaccion
        for (LineaPedido linea : request.getLineas()) {
            inventarioRepository.reservarStock(linea.getProductoId(), linea.getCantidad());
        }

        return pedidoRepository.save(pedido);
    }

    @Transactional(rollbackFor = {PedidoInvalidoException.class})
    public void actualizarEstado(Long id, String estado) {
        Pedido pedido = buscarPorId(id);
        pedido.setEstado(estado);
        // Dirty checking: Hibernate sincroniza automaticamente
    }
}
```

- La anotacion a nivel de clase establece un comportamiento por defecto para todos los metodos.
- Los metodos especificos sobrescriben la configuracion de clase segun sus necesidades.
- Las excepciones de tipo `RuntimeException` (y sus subclases) provocan rollback automatico. Para excepciones verificadas se debe usar `rollbackFor`.

## No repos ni controllers

Aplicar `@Transactional` en un repositorio puede parecer inofensivo, pero tiene consecuencias negativas:

```java
// MAL: transaccion en repositorio
@Repository
public class PedidoRepositoryImpl implements PedidoRepositoryCustom {

    @Transactional
    public void operacionCompleja() {
        // Si un servicio llama a este metodo junto a otro repositorio,
        // cada uno puede abrir su propia transaccion
    }
}
```

Si el servicio llama a dos metodos de repositorio, cada uno anotado con `@Transactional`, Spring crea dos transacciones separadas (o una propagacion anidada segun la configuracion de propagacion). Esto impide que el servicio las agrupe en una sola unidad atomica.

Los controladores tampoco deben gestionar transacciones:

```java
// MAL: @Transactional en controlador
@RestController
public class PedidoController {

    @Transactional
    @PostMapping("/pedidos")
    public ResponseEntity<PedidoResponse> crear(@RequestBody CrearPedidoRequest request) {
        // No debe haber logica transaccional en la capa web
    }
}
```

El controlador pertenece a la capa de presentacion. Mezclar responsabilidades transaccionales en esta capa impide reutilizar la logica de negocio desde otros puntos de entrada (eventos, jobs, tests) y acopla la gestion de transacciones al protocolo HTTP.

## readOnly=true para consultas

Marcar las consultas como `readOnly = true` permite a Hibernate y al motor de base de datos realizar optimizaciones significativas:

```java
@Transactional(readOnly = true)
public InformeDTO generarInformeMensual(LocalDate mes) {
    List<Pedido> pedidos = pedidoRepository.findByFechaBetween(
        mes.withDayOfMonth(1),
        mes.withDayOfMonth(mes.lengthOfMonth())
    );
    // Hibernate deshabilita el dirty checking, ahorrando ciclos
    // La base de datos puede evitar bloqueos exclusivos en las lecturas
    return new InformeDTO(pedidos);
}
```

Con `readOnly = true`:

- Hibernate desactiva el mecanismo de dirty checking (no necesita inspeccionar cada propiedad de las entidades gestionadas para detectar cambios).
- El EntityManager no genera flush automatico al final de la transaccion.
- La base de datos puede utilizar modos de lectura mas eficientes o bloqueos compartidos en lugar de exclusivos.
- Spring Data JPA hereda esta configuracion para los metodos de repositorio que se invoquen dentro del mismo contexto transaccional.

> No confundir `readOnly = true` con inmutabilidad. La entidad sigue siendo modificable en memoria, pero Hibernate ignorara los cambios al sincronizar con la base de datos. Si se necesita modificar, el metodo debe tener `@Transactional` sin `readOnly` o con `readOnly = false`.

## Timeout configurable

Las transacciones que bloquean recursos durante demasiado tiempo pueden degradar el rendimiento general del sistema. `@Transactional` permite configurar un timeout en segundos:

```java
@Transactional(timeout = 5)
public void procesarLotePagos(List<PagoRequest> pagos) {
    for (PagoRequest pago : pagos) {
        cuentaRepository.actualizarSaldo(pago.getCuentaId(), pago.getImporte());
    }
}
```

Si la transaccion no completa dentro del tiempo especificado, el EntityManager lanza `TransactionTimedOutException` y fuerza el rollback. El timeout se puede configurar globalmente en `application.yml`:

```yaml
spring:
  jpa:
    properties:
      javax:
        persistence:
          query:
            timeout: 5000  # milisegundos, afecta a cada consulta individual
      hibernate:
        connection:
          provider_disables_autocommit: true
```

> El timeout de transaccion no es lo mismo que el timeout de consulta JDBC (`javax.persistence.query.timeout`). El primero controla la duracion total de la transaccion; el segundo, el tiempo maximo de ejecucion de una sola consulta. Ambos deben configurarse segun los requisitos de la operacion.

## Propagacion y aislamiento

La anotacion `@Transactional` soporta atributos adicionales para controlar como se comporta la transaccion cuando se invoca desde otra transaccion existente:

```java
@Service
public class NotificacionService {

    // Crea una nueva transaccion independiente
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void registrarEventoAuditoria(EventoAuditoria evento) {
        eventoRepository.save(evento);
        // Se persiste incluso si la transaccion padre hace rollback
    }
}

@Service
public class PedidoService {

    @Transactional
    public void crearPedido(PedidoRequest request) {
        Pedido pedido = pedidoRepository.save(new Pedido(request));
        // Se ejecuta en transaccion separada
        notificacionService.registrarEventoAuditoria(
            new EventoAuditoria("PEDIDO_CREADO", pedido.getId())
        );
    }
}
```

- `Propagation.REQUIRED` (valor por defecto): reusa la transaccion existente o crea una nueva si no hay ninguna.
- `Propagation.REQUIRES_NEW`: suspende la transaccion actual y crea una nueva. Util para operaciones de auditoria que deben persistir independientemente del resultado de la transaccion principal.
- `Propagation.MANDATORY`: lanza excepcion si no hay una transaccion activa.
- `Propagation.NEVER`: lanza excepcion si hay una transaccion activa.

El nivel de aislamiento se configura con `isolation`:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void operacionSensible() { ... }
```

- `READ_COMMITTED` (por defecto en la mayoria de motores): evita lecturas sucias, pero permite lecturas no repetibles y fantasmas.
- `REPEATABLE_READ`: garantiza que si se lee una fila dos veces en la misma transaccion, se obtenga el mismo valor.
- `SERIALIZABLE`: nivel maximo, ejecuta las transacciones como si fueran secuenciales.

> En la mayoria de aplicaciones `READ_COMMITTED` ofrece el equilibrio adecuado entre consistencia y concurrencia. Niveles superiores aumentan la probabilidad de deadlocks y degradan el rendimiento. Solo deben usarse cuando el modelo de negocio lo exija explicitamente.
