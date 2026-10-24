# Programming Model

Spring WebFlux expone dos modelos de programacion para definir endpoints HTTP reactivos: el modelo basado en anotaciones, heredado de Spring MVC pero adaptado al paradigma reactivo, y el modelo funcional que ofrece un enfoque declarativo basado en funciones puras. Ambos coexisten dentro del mismo framework y comparten la misma infraestructura de ejecucion.

## Modelo basado en anotaciones

El modelo mas familiar para desarrolladores de Spring MVC se adapta directamente a WebFlux. Los controladores reactivos utilizan `@RestController` y devuelven `Mono` o `Flux` en lugar de tipos convencionales. El contenedor gestiona automaticamente la suscripcion al flujo y transmite los elementos al cliente conforme estan disponibles.

```java
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioController {

    private final UsuarioService usuarioService;

    public UsuarioController(UsuarioService usuarioService) {
        this.usuarioService = usuarioService;
    }

    @GetMapping("/{id}")
    public Mono<UsuarioDTO> obtenerPorId(@PathVariable Long id) {
        return usuarioService.buscarPorId(id)
            .map(this::toDTO);
    }

    @GetMapping
    public Flux<UsuarioDTO> listarTodos() {
        return usuarioService.listarTodos()
            .map(this::toDTO);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<UsuarioDTO> crear(@RequestBody @Valid Mono<UsuarioDTO> dto) {
        return dto.flatMap(usuarioService::guardar)
            .map(this::toDTO);
    }
}
```

La diferencia fundamental con Spring MVC radica en que el servidor no bloquea el hilo de peticion durante la resolucion del flujo. Cada elemento emitido por `Mono` o `Flux` se escribe en la respuesta de forma asincrona, permitiendo que el hilo del contenedor se libere inmediatamente despues de iniciar la operacion.

## Modelo funcional

El modelo funcional reemplaza las anotaciones con funciones de tipo de primera clase, proporcionando una alternativa mas declarativa y composable. Se compone de dos elementos principales: `RouterFunction` que define las rutas, y `HandlerFunction` que implementa la logica de negocio.

```java
@Configuration
public class RutasConfig {

    @Bean
    public RouterFunction<ServerResponse> rutas(UsuarioHandler handler) {
        return RouterFunctions.route()
            .path("/api/usuarios", builder -> builder
                .GET("/{id}", handler::obtenerPorId)
                .GET("", handler::listarTodos)
                .POST("", handler::crear)
                .PUT("/{id}", handler::actualizar)
                .DELETE("/{id}", handler::eliminar)
            )
            .filter((request, next) -> {
                // Filtro global aplicado a todas las rutas
                long inicio = System.currentTimeMillis();
                return next.handle(request)
                    .doOnSuccess(r -> log.info("Tiempo: {}ms",
                        System.currentTimeMillis() - inicio));
            })
            .build();
    }
}
```

Las `HandlerFunction` reciben un `ServerRequest` inmutable que encapsula los datos de la peticion HTTP y retornan un `Mono<ServerResponse>` con la respuesta construida de forma funcional.

```java
@Component
public class UsuarioHandler {

    private final UsuarioService usuarioService;

    public UsuarioHandler(UsuarioService usuarioService) {
        this.usuarioService = usuarioService;
    }

    public Mono<ServerResponse> obtenerPorId(ServerRequest request) {
        Long id = Long.valueOf(request.pathVariable("id"));
        return usuarioService.buscarPorId(id)
            .flatMap(usuario -> ServerResponse.ok()
                .bodyValue(toDTO(usuario)))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> crear(ServerRequest request) {
        return request.bodyToMono(UsuarioDTO.class)
            .flatMap(usuarioService::guardar)
            .flatMap(usuario -> ServerResponse
                .status(HttpStatus.CREATED)
                .bodyValue(toDTO(usuario)));
    }
}
```

La ventaja del modelo funcional reside en la composabilidad y testabilidad. Las funciones son testables de forma aislada sin necesidad de levantar el contenedor HTTP, y los filtros se aplican de forma declarativa sobre las cadenas de rutas.

## Server-Sent Events

WebFlux soporta nativamente **Server-Sent Events (SSE)**, un mecanismo de transmision unidireccional del servidor al cliente basado en HTTP. SSE es ideal para flujos de datos en tiempo real como notificaciones, actualizaciones de estado o监控 de eventos, donde el servidor emite datos periodicamente sin que el cliente solicite cada uno.

```java
@RestController
@RequestMapping("/api/events")
public class EventStreamController {

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<EventoDTO>> streamEventos() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(i -> ServerSentEvent.<EventoDTO>builder()
                .id(String.valueOf(i))
                .event("actualizacion")
                .data(new EventoDTO("Evento " + i, Instant.now()))
                .build());
    }
}
```

El modelo funcional tambien soporta SSE mediante `ServerResponse` con el tipo de medio adecuado:

```java
public Mono<ServerResponse> streamEventos(ServerRequest request) {
    Flux<ServerSentEvent<EventoDTO>> eventos = Flux.interval(Duration.ofSeconds(1))
        .map(i -> ServerSentEvent.<EventoDTO>builder()
            .id(String.valueOf(i))
            .data(new EventoDTO("Evento " + i, Instant.now()))
            .build());

    return ServerResponse.ok()
        .contentType(MediaType.TEXT_EVENT_STREAM)
        .body(eventos, new ParameterizedTypeReference<>() {});
}
```

> El cliente SSE reconecta automaticamente cuando la conexion se pierde, utilizando el header `Last-Event-ID` para retomar el flujo desde el punto de interrupcion. Esto lo hace mas robusto que WebSockets para escenarios de notificacion donde la conectividad puede ser inestable.

La combinacion de ambos modelos de programacion permite elegir el enfoque mas adecuado segun el contexto: anotaciones para APIs REST simples y el modelo funcional para composable y routing complejo.

