# Facade

Un subsistema complejo expone decenas de clases, metodos y dependencias que el cliente debe conocer y orquestar para realizar una tarea simple. El patron Facade introduce un objeto que ofrece una interfaz unificada y simplificada, ocultando la complejidad interna y reduciendo el acoplamiento entre el cliente y los componentes del subsistema.

```java
public class SistemaInterno {
    public void conectar() { /* ... */ }
    public void autenticar(String token) { /* ... */ }
    public void enviarPayload(String xml) { /* ... */ }
    public void verificarEstado() { /* ... */ }
    public void cerrar() { /* ... */ }
}

public class FachadaServicioExterno {

    private final SistemaInterno sistema;
    private final Auditoria auditoria;

    public void enviar(String token, String xml) {
        sistema.conectar();
        sistema.autenticar(token);
        sistema.enviarPayload(xml);
        if (!sistema.verificarEstado()) {
            auditoria.registrarError(xml);
            throw new EnvioFallidoException();
        }
        sistema.cerrar();
    }
}
```

El cliente ya no necesita conocer `SistemaInterno`, ni saber el orden correcto de invocacion, ni gestionar errores: la fachada encapsula ese conocimiento. La complejidad del subsistema sigue existiendo, pero queda detras de una frontera delgada.

## Fachada en la capa de servicio

El service layer descrito anteriormente es, en esencia, una fachada. Cada servicio expone unos pocos metodos que representan casos de uso completos, mientras que internamente coordina repositorios, servicios de dominio, transformadores de DTO y posiblemente llamadas a sistemas externos. El controlador REST no conoce `ClienteRepository`, `ProductoRepository`, `CalculadorImpuestos` ni la logica de creacion de `DetalleFactura`: solo conoce `FacturacionService` y su metodo `emitirFactura()`.

La fachada no se limita a delegar ciegamente; puede realizar tareas transversales como:

- Validar parametros de entrada antes de invocar al subsistema
- Transformar objetos entre la representacion externa (DTO) e interna (entidades)
- Manejar excepciones del subsistema y convertirlas en excepciones de aplicacion
- Registrar auditoria o metricas alrededor de cada operacion

```java
@RestController
public class FacturaController {
    private final FacturacionService facturacionService;

    @PostMapping("/facturas")
    public ResponseEntity<FacturaDTO> emitir(@RequestBody @Valid FacturaRequest request) {
        FacturaDTO factura = facturacionService.emitirFactura(
                request.clienteId(), request.lineas());
        return ResponseEntity.status(HttpStatus.CREATED).body(factura);
    }
}
```

## Diseno de la fachada

Una fachada bien disenada debe cumplir dos condiciones. Primera, ofrecer metodos con un nivel de granularidad adecuado al cliente: ni tan finos que el cliente tenga que llamar a varios metodos para completar una accion, ni tan gruesos que sea inflexible. Segunda, no filtrar la complejidad interna: si el cliente necesita conocer las clases del subsistema para usar la fachada, esta no esta cumpliendo su proposito.

No todas las operaciones del subsistema necesitan estar expuestas en la fachada. Solo aquellas que tienen sentido para el cliente. Las operaciones internas de mantenimiento, configuracion o inicializacion permanecen ocultas.

> Una fachada no es un wrapper. El wrapper envuelve una unica clase para cambiar su interfaz; la fachada abstrae un subsistema completo compuesto por multiples clases que colaboran.

## Relacion con otros patrones

- **Service Layer:** es un tipo de fachada, pero con responsabilidades transaccionales y de orquestacion de logica de negocio.
- **Adapter:** adapta la interfaz de una clase concreta a otra esperada por el cliente. La fachada simplifica todo un subsistema, no adapta una interfaz concreta.
- **Mediator:** coordina la comunicacion entre multiples objetos. La fachada unidireccionalmente simplifica el acceso al subsistema; no gestiona interacciones entre pares.

Una fachada puede internamente utilizar adaptadores, proxies o mediators para cumplir su cometido. La fachada es un punto de entrada, no un patrón de comunicacion interna.
