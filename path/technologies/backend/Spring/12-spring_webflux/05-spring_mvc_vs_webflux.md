# Spring MVC vs WebFlux

Spring MVC y Spring WebFlux son dos paradigmas dentro del ecosistema Spring que resuelven el problema de servir contenido HTTP, pero con filosofias fundamentalmente distintas. Spring MVC se basa en el modelo request-response sobre Servlet API con un hilo dedicado por cada peticion entrante, mientras que WebFlux adopta un modelo reactivo no-bloqueante sobre Reactive Streams. Elegir uno u otro no es una cuestion de preferencia personal sino de requisitos tecnicos del dominio.

## Modelo de ejecucion

En Spring MVC, cada peticion HTTP se asigna a un hilo del pool del contenedor Servlet (por defecto Tomcat). Ese hilo permanece asignado durante toda la duracion de la peticion: desde la recepcion del request hasta la escritura completa de la respuesta. Si la peticion realiza una consulta a base de datos que tarda 200ms, el hilo queda bloqueado durante esos 200ms sin hacer nada util. Este modelo es intuitivo y sencillo de depurar, pero escala de forma limitada en escenarios de alta concurrencia con operaciones de E/S.

WebFlux reemplaza este modelo con un motor de ejecucion reactivo basado en hilos de E/S y un pool de workers configurable. Un numero reducido de hilos puede gestionar miles de conexiones concurrentes porque ninguno de ellos permanece bloqueado esperando una respuesta de red o base de datos. Cuando una operacion de E/S se inicia, el hilo se libera para procesar otra peticion y recibe una notificacion asincrona cuando la operacion completa.

```yaml
# Configuracion de Netty en WebFlux
server:
  netty:
    connection-timeout: 10s
    idle-timeout: 60s
    max-keep-alive-requests: 100
```

## Cuando elegir WebFlux

WebFlux es la opcion adecuada cuando la aplicacion se caracteriza por:

- Alta concurrencia con un numero elevado de conexiones simultaneas (miles o decenas de miles)
- Operaciones de E/S latentes como llamadas a servicios externos, bases de datos o APIs de terceros
- Necesidad de streaming de datos en tiempo real (SSE, WebSockets)
- Arquitectura de microservicios donde la comunicacion entre servicios es intensiva

```java
// WebFlux: el hilo no se bloquea durante la espera
@GetMapping("/usuarios")
public Flux<UsuarioDTO> listar() {
    return usuarioRepository.findAll() // Consulta reactiva no-bloqueante
        .map(this::toDTO)
        .delayElements(Duration.ofMillis(50)); // Simula latencia
}

// Spring MVC: el hilo se bloquea durante toda la operacion
@GetMapping("/usuarios")
public List<UsuarioDTO> listar() {
    return usuarioRepository.findAll() // Consulta bloqueante
        .stream()
        .map(this::toDTO)
        .collect(Collectors.toList());
}
```

## Cuándo mantenerse en Spring MVC

Spring MVC sigue siendo la eleccion correcta para la mayoria de aplicaciones web convencionales:

- APIs REST con baja a moderada concurrencia donde el bottleneck no es la E/S
- Aplicaciones que dependen fuertemente del ecosistema Servlet (filters, listeners, JSP)
- Equipos con experiencia consolidada en el modelo bloqueante
- Procesamiento intensivo de CPU donde la concurrencia no es el factor limitante
- Integracion con librerias que asumen ejecucion bloqueante

## Rendimiento y benchmarks

En terminos de rendimiento puro, la diferencia entre ambos modelos depende del escenario. Para operaciones CPU-bound (computacion pesada sin E/S), Spring MVC puede ser igual o mas rapido porque el overhead de la programacion reactiva no se compensa con la ganancia en concurrencia. Para operaciones I/O-bound, WebFlux puede manejar significativamente mas peticiones concurrentes con menos recursos del sistema.

Un benchmark representativo mostraria que una aplicacion MVC con 200 hilos puede manejar aproximadamente 200 peticiones concurrentes con latencia estable, mientras que WebFlux con 8 hilos puede manejar miles de peticiones concurrentes con latencia similar siempre que las operaciones de E/S sean no-bloqueantes.

> WebFlux no es inherentemente mas rapido que Spring MVC. Es mas eficiente en el uso de recursos del sistema bajo carga concurrente con operaciones de E/S. Si la aplicacion no tiene ese perfil de uso, la complejidad adicional de la programacion reactiva no aporta beneficios medibles.

## Endpoint funcional vs anotaciones

WebFlux ofrece los `RouterFunction` y `HandlerFunction` como alternativa al modelo basado en anotaciones. Este modelo funcional permite definir rutas de forma declarativa sin reflexion ni anotaciones, lo que puede mejorar el rendimiento en tiempo de arranque (especialmente relevante en native images de GraalVM).

```java
// WebFlux funcional
RouterFunction<ServerResponse> rutas = route()
    .GET("/api/usuarios/{id}", handler::obtener)
    .GET("/api/usuarios", handler::listar)
    .POST("/api/usuarios", handler::crear)
    .build();

// Spring MVC equivalente
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioController {
    @GetMapping("/{id}")
    public Mono<UsuarioDTO> obtener(@PathVariable Long id) { ... }
    @GetMapping
    public Flux<UsuarioDTO> listar() { ... }
    @PostMapping
    public Mono<UsuarioDTO> crear(@RequestBody UsuarioDTO dto) { ... }
}
```

Ambos modelos coexisten en la misma aplicacion y comparten infraestructura comun. No es necesario migrar toda la aplicacion a WebFlux de forma atomica; es posible incluir endpoints reactivos junto a endpoints MVC en la misma aplicacion si se utiliza un servidor compatible con ambos paradigmas.

