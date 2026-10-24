# Propagacion de transacciones

La propagacion determina como se comporta un metodo transaccional cuando es invocado desde otro metodo que ya tiene una transaccion activa. Spring define siete niveles de propagacion a traves del enumerado `Propagation`, cada uno con un proposito especifico que permite disenar el comportamiento transaccional de la aplicacion con precision.

## REQUIRED (comportamiento por defecto)

Usa la transaccion actual si existe; si no, crea una nueva. Es el valor por defecto y el mas utilizado.

```
Metodo A (@Transactional)  -->  Metodo B (@Transactional REQUIRED)
    Transaccion T1              Transaccion T1 (reutiliza)
                                    commit T1
    commit T1
```

```java
@Service
public class PedidoService {

    @Autowired
    private FacturaService facturaService;

    @Transactional(propagation = Propagation.REQUIRED)
    public void crearPedido(Pedido pedido) {
        pedidoRepository.save(pedido);
        facturaService.generarFactura(pedido);   // Misma transaccion
    }
}

@Service
public class FacturaService {

    @Transactional(propagation = Propagation.REQUIRED)
    public void generarFactura(Pedido pedido) {
        facturaRepository.save(new Factura(pedido));
    }
}
```

Si `generarFactura` lanza una excepcion, toda la transaccion se revierte, incluyendo `pedidoRepository.save(pedido)`. No hay forma de confirmar parcialmente.

## REQUIRES_NEW

Suspende la transaccion actual (si existe) y crea una nueva transaccion independiente. La transaccion suspendida se reanuda cuando la nueva transaccion finaliza.

```
Metodo A (@Transactional)  -->  Metodo B (@Transactional REQUIRES_NEW)
    Transaccion T1               Suspende T1
                                     Crea T2
                                     commit T2
                                     Reanuda T1
    commit T1
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void registrarAuditoria(Evento evento) {
    auditoriaRepository.save(evento);
    // Esta transaccion es independiente
    // Se confirma aunque la transaccion padre falle
}
```

> `REQUIRES_NEW` es util para auditoria, logging o envio de notificaciones donde cada operacion debe persistir independientemente del resultado de la transaccion principal. Sin embargo, tiene un coste: cada nueva transaccion implica una conexion JDBC separada y recursos adicionales en la base de datos.

## NESTED

Crea una transaccion anidada dentro de la transaccion actual. Utiliza savepoints de JDBC para permitir rollbacks parciales.

```java
@Transactional(propagation = Propagation.NESTED)
public void procesarItem(ItemPedido item) {
    itemRepository.save(item);
    // Si falla, solo se revierte este savepoint
}

// Uso tipico: procesar una lista de items independientes
@Transactional
public void procesarPedido(Pedido pedido) {
    for (ItemPedido item : pedido.getItems()) {
        try {
            procesarItem(item);  // NESTED - cada item es un savepoint
        } catch (Exception e) {
            log.error("Item fallo, continuando con el siguiente", e);
            // Solo el item fallido se revierte
        }
    }
}
```

NESTED solo funciona con `DataSourceTransactionManager` (JDBC). Con JPA (`JpaTransactionManager`), NESTED se comporta como REQUIRED porque JPA no soporta savepoints a nivel de EntityManager.

> A diferencia de REQUIRES_NEW, NESTED no es una transaccion independiente. Si la transaccion principal se revierte, todos los savepoints anidados tambien se revierten, independientemente de que se hayan confirmado individualmente.

## SUPPORTS

Si existe una transaccion activa, se une a ella. Si no, ejecuta el metodo sin transaccion.

```java
@Transactional(propagation = Propagation.SUPPORTS)
public List<Producto> buscarProductos(String nombre) {
    return productoRepository.findByNombre(nombre);
}
```

SUPPORTS es util para metodos que pueden ejecutarse tanto en contextos transaccionales como no transaccionales. Tipicamente se usa en metodos de consulta que no requieren una transaccion pero se benefician de ella si ya existe.

## NOT_SUPPORTED

Suspende la transaccion actual (si existe) y ejecuta el metodo fuera de cualquier transaccion.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void exportarReporte(Reporte reporte) {
    // Operacion larga que no debe mantener bloqueos
    generarCsv(reporte);
    enviarPorEmail(reporte);
}
```

NOT_SUPPORTED es adecuado para operaciones largas que no deben retener conexiones del pool ni bloqueos de base de datos.

## MANDATORY

Exige que exista una transaccion activa. Si no hay ninguna, lanza una excepcion `IllegalTransactionStateException`.

```java
@Transactional(propagation = Propagation.MANDATORY)
public void actualizarStock(Producto producto) {
    // Solo puede llamarse desde un contexto transaccional
    productoRepository.save(producto);
}
```

MANDATORY se usa cuando un metodo debe ejecutarse obligatoriamente dentro de una transaccion iniciada por el llamante. Es una forma de documentar y validar la dependencia transaccional.

## NEVER

Exige que NO exista una transaccion activa. Si hay alguna, lanza una excepcion `IllegalTransactionStateException`.

```java
@Transactional(propagation = Propagation.NEVER)
public void inicializarCache() {
    cacheManager.limpiar();
    cacheManager.cargarDatos();
}
```

NEVER se utiliza para operaciones que explicitamente no deben participar en ninguna transaccion, como inicializaciones, limpieza de cache o tareas de mantenimiento.

## Diagrama de propagacion

```
Llamada desde:   Sin transaccion   |   Con transaccion
------------------------------------+-------------------
REQUIRED         Crea nueva         |   Se une a la actual
REQUIRES_NEW     Crea nueva         |   Suspende actual, crea nueva
NESTED           Crea nueva         |   Crea savepoint
SUPPORTS         Sin transaccion   |   Se une a la actual
NOT_SUPPORTED    Sin transaccion   |   Suspende actual
MANDATORY        Excepcion         |   Se une a la actual
NEVER            Sin transaccion   |   Excepcion
```

## Consideraciones practicas

La propagacion REQUIRES_NEW y NESTED son las que mas implicaciones tienen en el diseno de la aplicacion:

- **REQUIRES_NEW** consume una conexion JDBC adicional del pool mientras la transaccion original esta suspendida. Si hay muchas llamadas anidadas, se puede agotar el pool rapidamente.
- **NESTED** no funciona correctamente con todos los gestores de transaccion. Con JPA, no hay soporte nativo de savepoints aplicados al contexto de persistencia, por lo que el comportamiento real es equivalente a REQUIRED.
- **MANDATORY y NEVER** son utiles para forzar contratos entre capas, pero rara vez son necesarios en aplicaciones tipicas. Su uso excesivo acopla las capas innecesariamente.

> La eleccion del nivel de propagacion debe estar alineada con el modelo de dominio y los requisitos de atomicidad. No todos los metodos necesitan una transaccion independiente. Usar REQUIRES_NEW indiscriminadamente es una fuente comun de problemas de rendimiento y agotamiento del pool de conexiones.
