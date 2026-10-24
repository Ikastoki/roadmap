# Mejores practicas de transacciones

La correcta gestion de transacciones en Spring requiere mas que conocer las anotaciones y propagaciones: implica definir donde colocar las transacciones, que nivel de aislamiento usar, como manejar excepciones y cuales configuraciones optimizar para produccion. Estas practicas se derivan de problemas reales encontrados en aplicaciones en entornos productivos y representan el consenso de la comunidad de Spring.

## Colocacion de @Transactional en la capa de servicio

La anotacion `@Transactional` debe colocarse preferiblemente en la capa de servicio, no en controllers ni en repositories. Los controllers son componentes de presentacion que delegan la logica de negocio a los servicios. Si la transaccion se define en el controller, la capa de servicio pierde la oportunidad de reutilizar metodos transaccionales en diferentes contextos, y el ciclo de vida de la transaccion queda atado al request HTTP en lugar de a la operacion de negocio.

```java
// CORRECTO: transaccion en la capa de servicio
@Service
public class PedidoService {

    @Transactional
    public Pedido crearPedido(PedidoDTO dto) {
        Pedido pedido =PedidoMapper.toEntity(dto);
        pedidoRepository.save(pedido);
        inventarioService.descontar(pedido.getItems());
        return pedido;
    }
}

// INCORRECTO: transaccion en el controller
@RestController
public class PedidoController {

    @PostMapping("/pedidos")
    @Transactional  // NO hacer esto
    public ResponseEntity<Pedido> crear(@RequestBody PedidoDTO dto) {
        return ResponseEntity.ok(pedidoService.crearPedido(dto));
    }
}
```

En Spring Data, los repositories ya gestionan transacciones individuales por defecto. Un metodo `save()` se ejecuta dentro de su propia transaccion. Si el servicio anota con `@Transactional`, el repository se ejecuta dentro de la transaccion del servicio, creando un comportamiento correcto de transaccion unica. Sin `@Transactional` en el servicio, cada llamada al repository seria una transaccion independiente, lo que puede causar inconsistencias.

```java
// Sin @Transactional en el servicio: cada operacion es una transaccion separada
public void crearPedido(Pedido pedido) {
    pedidoRepository.save(pedido);          // transaccion 1 - commitea aqui
    inventarioService.descontarStock(...);  // transaccion 2 - si falla aqui,
                                             // el pedido ya esta guardado
}
```

## No colocar @Transactional en repositories

Los repositories en Spring Data ya gestionan transacciones individuales. Anotar un metodo de repository con `@Transactional` puede causar efectos inesperados, especialmente con `readOnly=true`. Spring Data JPA marca como read-only por defecto los metodos `find*`, `get*`, `list*`, `stream*` y `count*`, y como escritura los metodos `save`, `delete` y `remove`.

Si un servicio necesita agrupar multiples operaciones de repository en una unica transaccion, la anotacion debe ir en el servicio, no en el repository.

```java
// Repository: Spring Data maneja las transacciones internamente
public interface PedidoRepository extends JpaRepository<Pedido, Long> {
    List<Pedido> findByEstado(EstadoPedido estado);  // read-only automatica
    Pedido save(Pedido pedido);                       // transaccion de escritura
}

// Servicio: agrupa operaciones en una transaccion
@Service
public class PedidoService {

    @Transactional
    public void procesarPedidosPendientes() {
        List<Pedido> pendientes = pedidoRepository.findByEstado(EstadoPedido.PENDIENTE);
        for (Pedido pedido : pendientes) {
            pedido.setEstado(EstadoPedido.PROCESADO);
            pedidoRepository.save(pedido);
        }
    }
}
```

## readOnly=true para operaciones de solo lectura

Todo metodo que solo lea datos y no modifique el estado de la base de datos debe marcarse con `@Transactional(readOnly = true)`. Esto no es solo semantico: Hibernate puede deshabilitar el dirty checking, no adquirir locks de escritura y optimizar el flush del persistence context. En JPA, la transaccion read-only evita que el persistence context realice una sincronizacion al final del metodo.

```java
@Service
public class BusquedaService {

    @Transactional(readOnly = true)
    public List<Usuario> buscarUsuarios(String nombre) {
        return usuarioRepository.findByNombreContaining(nombre);
    }

    @Transactional(readOnly = true)
    public Optional<Usuario> encontrarPorId(Long id) {
        return usuarioRepository.findById(id);
    }

    @Transactional(readOnly = true)
    public long contarActivos() {
        return usuarioRepository.countByActivoTrue();
    }
}
```

## Propagacion REQUIRED para escritura

Para metodos de escritura, la propagacion `REQUIRED` (la por defecto) es la correcta en la mayoria de casos. Permite que multiples operaciones de escritura se agrupen en una sola transaccion, manteniendo la atomicidad. No es necesario especificar `@Transactional` sin atributos, ya que `REQUIRED` es el default.

```java
@Service
public class UserService {

    // REQUIRED es el default, equivalente a @Transactional sin atributos
    @Transactional
    public void actualizarPerfil(Long userId, PerfilDTO perfil) {
        Usuario usuario = usuarioRepository.findById(userId).orElseThrow();
        usuario.setNombre(perfil.getNombre());
        usuario.setEmail(perfil.getEmail());
        usuarioRepository.save(usuario);
        historialService.registrarCambio(userId, "Perfil actualizado");
    }
}
```

## REQUIRES_NEW para logs de auditoria

Cuando se necesita registrar logs de auditoria o eventos que deben persistirse independientemente de si la operacion principal tiene exito o falla, se usa `REQUIRES_NEW`. Esta propagacion crea una transaccion separada que se commitea incluso si la transaccion padre hace rollback.

```java
@Service
public class AuditoriaService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void registrarEvento(String evento, String detalles) {
        LogAuditoria log = new LogAuditoria(evento, detalles, Instant.now());
        logRepository.save(log);
        // Se commitea aunque la transaccion que llamo haga rollback
    }
}

@Service
public class PedidoService {

    @Transactional
    public Pedido crearPedido(Pedido pedido) {
        try {
            Pedido guardado = pedidoRepository.save(pedido);
            auditoriaService.registrarEvento("PEDIDO_CREADO", guardado.getId().toString());
            return guardado;
        } catch (Exception e) {
            auditoriaService.registrarEvento("PEDIDO_ERROR", e.getMessage());
            throw e;
        }
    }
}
```

## Timeout configurado

Siempre es buena practica configurar un timeout en las transacciones, especialmente en operaciones de escritura que puedan bloquear recursos. Un timeout demasiado largo puede causar deadlocks; uno demasiado corto puede fallar en operaciones legitimas.

```java
@Service
public class ReporteService {

    @Transactional(readOnly = true, timeout = 60)
    public Reporte generarReporteCompleto() {
        // Operacion de lectura compleja que puede tardar
        // Si supera 60 segundos, se cancela
    }

    @Transactional(timeout = 15)
    public void procesarPago(Pago pago) {
        // Si no se procesa en 15 segundos, se revierte
    }
}
```

## Exception Handling y Transacciones

El manejo de excepciones dentro de transacciones es una de las fuentes mas comunes de bugs. La regla fundamental es: las `RuntimeException` provocan rollback; las checked exceptions no (por defecto). Para evitar sorpresas, se recomienda especificar `rollbackFor = Exception.class` o manejar consistentemente las excepciones.

```java
@Service
public class PagoService {

    @Transactional(rollbackFor = Exception.class)
    public void procesarPago(Pago pago) throws PagoRechazadoException {
        try {
            pasarelaPago.cobrar(pago);
            pago.setEstado(EstadoPago.COMPLETADO);
        } catch (PasarelaException e) {
            pago.setEstado(EstadoPago.FALLIDO);
            throw new PagoRechazadoException("Pago rechazado: " + e.getMessage());
        } finally {
            pagoRepository.save(pago); // esta operacion tambien se revierte
        }
    }
}
```

> Un error comun es intentar hacer rollback manualmente dentro de un metodo `@Transactional` usando `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`. Esto es valido pero innecesario si se lanza la excepcion adecuada. Reserve el rollback manual para casos donde no se desea lanzar una excepcion.

