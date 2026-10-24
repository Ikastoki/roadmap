# Service Layer

Una aplicacion empresarial rara vez consiste en una simple operacion de lectura o escritura sobre una base de datos. Por lo general, cada caso de uso involucra multiples pasos: validar datos, aplicar reglas de negocio, consultar varios repositorios, modificar entidades y coordinar el resultado final. El patron Service Layer encapsula toda esa orquestacion en un conjunto de objetos servicos que actuan como la frontera explicita entre la capa de presentacion y la capa de dominio o persistencia.

Un servicio representa un caso de uso completo. Su interfaz --ya sea una interfaz Java formal o simplemente los metodos publicos de una clase-- define las operaciones atomicas que el sistema ofrece al exterior. Cada metodo recibe DTOs o parametros simples, ejecuta la logica necesaria y devuelve el resultado, ocultando al cliente los detalles de como se obtienen, persisten o transforman los datos.

```java
public interface FacturacionService {
    FacturaDTO emitirFactura(Long clienteId, List<LineaFacturaDTO> lineas);
    void anularFactura(Long facturaId, String motivo);
    List<FacturaDTO> listarFacturasPorCliente(Long clienteId);
}

public class FacturacionServiceImpl implements FacturacionService {

    private final ClienteRepository clienteRepository;
    private final FacturaRepository facturaRepository;
    private final ProductoRepository productoRepository;
    private final CalculadorImpuestos calculadorImpuestos;

    public FacturacionServiceImpl(ClienteRepository clienteRepository,
                                   FacturaRepository facturaRepository,
                                   ProductoRepository productoRepository,
                                   CalculadorImpuestos calculadorImpuestos) {
        this.clienteRepository = clienteRepository;
        this.facturaRepository = facturaRepository;
        this.productoRepository = productoRepository;
        this.calculadorImpuestos = calculadorImpuestos;
    }

    @Override
    @Transactional
    public FacturaDTO emitirFactura(Long clienteId, List<LineaFacturaDTO> lineas) {
        Cliente cliente = clienteRepository.findById(clienteId)
                .orElseThrow(() -> new ClienteNoEncontradoException(clienteId));

        List<DetalleFactura> detalles = new ArrayList<>();
        for (LineaFacturaDTO linea : lineas) {
            Producto producto = productoRepository.findById(linea.getProductoId())
                    .orElseThrow(() -> new ProductoNoEncontradoException(linea.getProductoId()));
            detalles.add(new DetalleFactura(producto, linea.getCantidad()));
        }

        BigDecimal subtotal = detalles.stream()
                .map(DetalleFactura::calcularSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal impuestos = calculadorImpuestos.aplicar(subtotal);
        BigDecimal total = subtotal.add(impuestos);

        Factura factura = new Factura(cliente, detalles, impuestos, total);
        factura = facturaRepository.save(factura);

        return FacturaDTO.desdeEntidad(factura);
    }
}
```

## Responsabilidades transaccionales

El service layer es el lugar natural para demarcar las transacciones. Cada metodo de servicio que modifique el estado del sistema debe ser atomico: o se ejecuta completamente o no deja rastro. En el ecosistema Spring, la anotacion `@Transactional` sobre el metodo de servicio asegura que el EntityManager abra una transaccion al inicio, ejecute todas las operaciones dentro del mismo contexto transaccional y haga commit o rollback segun el resultado.

La transaccion no solo protege la integridad de la base de datos, sino que ademas garantiza que el EntityManager opere con un contexto de persistencia coherente. Durante la ejecucion del metodo, todas las entidades recuperadas estan gestionadas (managed), los cambios se detectan automaticamente (dirty checking) y cualquier excepcion no recuperable provoca un rollback implicito.

```java
@Transactional(readOnly = true)
public List<FacturaDTO> listarFacturasPorCliente(Long clienteId) {
    return facturaRepository.findByClienteId(clienteId).stream()
            .map(FacturaDTO::desdeEntidad)
            .toList();
}
```

La distincion entre metodos de solo lectura y metodos de escritura es importante. Marcar una operacion como `readOnly = true` permite al proveedor JPA optimizar el acceso (por ejemplo, desactivando el dirty checking) y en algunos motores de base de datos evita la adquisicion de locks innecesarios.

## Coordinacion entre repositorios

Un servicio no se limita a invocar un unico repositorio. Con frecuencia debe coordinar varios accesos a datos, validaciones y llamadas a sistemas externos. Por ejemplo, al registrar un pedido se debe comprobar el stock disponible, descontar las unidades, crear el pedido en base de datos y, finalmente, enviar una notificacion al almacen. Cada una de estas operaciones pertenece a un repositorio o servicio distinto, y es el service layer quien las orquesta en el orden correcto y dentro de la misma transaccion.

```java
@Transactional
public PedidoDTO crearPedido(PedidoRequest request) {
        Cliente cliente = clienteRepository.findById(request.clienteId())
                .orElseThrow();

        Pedido pedido = new Pedido(cliente);

        for (ItemRequest item : request.items()) {
            Producto producto = productoRepository.findByIdConLock(item.productoId())
                    .orElseThrow();
            if (producto.getStock() < item.cantidad()) {
                throw new StockInsuficienteException(producto, item.cantidad());
            }
            producto.descontarStock(item.cantidad());
            pedido.agregarLinea(producto, item.cantidad(), producto.getPrecio());
        }

        pedido = pedidoRepository.save(pedido);
        notificacionService.notificarAlmacen(pedido);

        return PedidoDTO.desdeEntidad(pedido);
}
```

## Servicios transaccionales frente a servicios de dominio

Conviene distinguir dos tipos de servicio. Los **servicios de aplicacion** (o service layer propiamente dicho) orquestan casos de uso, manejan transacciones y actuan como fachada ante el exterior. Los **servicios de dominio**, en cambio, encapsulan logica de negocio que no pertenece naturalmente a una sola entidad: por ejemplo, un `CalculadorImpuestos` o un `TransferenciaService` que mueve dinero entre dos cuentas. Los primeros dependen de infraestructura (repositorios, frameworks transaccionales) y pertenecen a la capa de aplicacion; los segundos forman parte del modelo de dominio y carecen de efectos secundarios tecnologicos.

El service layer no debe contener logica de negocio que pueda expresarse en las propias entidades. Su responsabilidad es la coordinacion, no la decision. Cuando una entidad puede calcular un total o validar una regla, ese codigo pertenece a la entidad, no al servicio.

```java
// Bien: el servicio solo orquesta
public void transferir(String origenIban, String destinoIban, BigDecimal monto) {
    Cuenta origen = cuentaRepository.findByIban(origenIban);
    Cuenta destino = cuentaRepository.findByIban(destinoIban);
    origen.debitar(monto);    // logica de negocio en la entidad
    destino.acreditar(monto); // logica de negocio en la entidad
    cuentaRepository.save(origen);
    cuentaRepository.save(destino);
}

// Mal: la logica de debito esta en el servicio
public void transferir(String origenIban, String destinoIban, BigDecimal monto) {
    Cuenta origen = cuentaRepository.findByIban(origenIban);
    Cuenta destino = cuentaRepository.findByIban(destinoIban);
    if (origen.getSaldo().compareTo(monto) < 0) {
        throw new SaldoInsuficienteException();
    }
    origen.setSaldo(origen.getSaldo().subtract(monto)); // logica fuera de lugar
    destino.setSaldo(destino.getSaldo().add(monto));
}
```

## Inyeccion de dependencias y testabilidad

Una de las ventajas fundamentales del service layer es que sus implementaciones pueden depender exclusivamente de interfaces (repositorios, otros servicios, factories). Esto permite sustituir cualquier dependencia por un doble de prueba (mock o stub) sin modificar el servicio. La inyeccion de dependencias --ya sea mediante constructor, como en el ejemplo anterior, o mediante un contenedor como Spring-- hace que los servicios sean unidades facilmente verificables de forma aislada.

```java
@Test
void testEmitirFactura() {
    var cliente = new Cliente(1L, "Empresa X");
    var producto = new Producto(10L, "Laptop", new BigDecimal("1000"));
    var lineas = List.of(new LineaFacturaDTO(10L, 2));

    when(clienteRepository.findById(1L)).thenReturn(Optional.of(cliente));
    when(productoRepository.findById(10L)).thenReturn(Optional.of(producto));
    when(calculadorImpuestos.aplicar(any())).thenReturn(new BigDecimal("210"));

    FacturaDTO resultado = facturacionService.emitirFactura(1L, lineas);

    assertNotNull(resultado);
    // ...
}
```
