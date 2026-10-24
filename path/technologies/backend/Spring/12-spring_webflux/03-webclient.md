# WebClient

`WebClient` es el cliente HTTP no-bloqueante y reactivo introducido en Spring 5 como reemplazo moderno de `RestTemplate`. A diferencia de su predecesor, `WebClient` se construye sobre la arquitectura reactiva de Project Reactor y permite comunicacion asincrona con servicios externos sin Consumir un hilo del pool durante la espera de la respuesta. Es el componente recomendado para la interaccion HTTP dentro de aplicaciones WebFlux y tambien es perfectamente utilizable en aplicaciones Spring MVC tradicionales.

## Construccion del cliente

`WebClient` se obtiene a traves de `WebClient.Builder`, que proporciona un builder preconfigurado con opciones de serializacion, timeouts y filtros. La creacion del builder es un proceso costoso en terminos de recursos, por lo que se recomienda inyectarlo como un bean singleton y reutilizarlo en toda la aplicacion.

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
            .baseUrl("https://api.ejemplo.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .filter((request, next) -> {
                log.info("Request: {} {}", request.method(), request.url());
                return next.exchange(request)
                    .doOnNext(response -> log.info("Status: {}", response.statusCode()));
            });
    }

    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder.build();
    }
}
```

Los filtros se encadenan de forma similar a los filtros de un `WebFilter` en el lado del servidor. Cada filtro recibe la `ClientRequest` y la funcion `next` que invoca al siguiente filtro en la cadena, permitiendo loggear, autenticar o transformar peticiones y respuestas.

## Operaciones GET

Una operacion GET basica recupera un recurso del servidor remoto y retorna un `Mono` con el objeto deserializado. `WebClient` soporta tanto la forma reactiva (usando `retrieve`) como la forma bloqueante (usando `block()`).

```java
@Service
public class ExternalApiClient {

    private final WebClient webClient;

    public ExternalApiClient(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("https://api.ejemplo.com").build();
    }

    public Mono<UsuarioDTO> obtenerUsuario(Long id) {
        return webClient.get()
            .uri("/usuarios/{id}", id)
            .retrieve()
            .bodyToMono(UsuarioDTO.class);
    }

    public Flux<UsuarioDTO> listarUsuarios() {
        return webClient.get()
            .uri("/usuarios")
            .retrieve()
            .bodyToFlux(UsuarioDTO.class);
    }
}
```

Para respuestas que incluyen metadatos HTTP (headers, status code), se utiliza `exchange()` que retorna un `ClientResponse`:

```java
public Mono<ResponseEntity<UsuarioDTO>> obtenerConMetadatos(Long id) {
    return webClient.get()
        .uri("/usuarios/{id}", id)
        .exchangeToMono(response -> response.bodyToMono(UsuarioDTO.class)
            .map(body -> ResponseEntity.status(response.statusCode())
                .headers(response.headers().asHttpHeaders())
                .body(body)));
}
```

## Operaciones POST, PUT y DELETE

Las operaciones de escritura siguen el mismo patron, pero incluyen un cuerpo de peticion (`body`) con los datos a enviar:

```java
public Mono<UsuarioDTO> crear(UsuarioDTO nuevo) {
    return webClient.post()
        .uri("/usuarios")
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(nuevo)
        .retrieve()
        .bodyToMono(UsuarioDTO.class);
}

public Mono<UsuarioDTO> actualizar(Long id, UsuarioDTO actualizado) {
    return webClient.put()
        .uri("/usuarios/{id}", id)
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(actualizado)
        .retrieve()
        .bodyToMono(UsuarioDTO.class);
}

public Mono<Void> eliminar(Long id) {
    return webClient.delete()
        .uri("/usuarios/{id}", id)
        .retrieve()
        .bodyToMono(Void.class)
        .then();
}
```

Para envios que requierenStreaming de datos, `body()` acepta `Flux` como cuerpo de peticion, enviando los elementos conforme estan disponibles:

```java
public Mono<BatchResponse> enviarLote(Flux<RegistroDTO> registros) {
    return webClient.post()
        .uri("/api/lote")
        .contentType(MediaType.APPLICATION_NDJSON)
        .body(registros, RegistroDTO.class)
        .retrieve()
        .bodyToMono(BatchResponse.class);
}
```

## Manejo de errores

El manejo de errores en `WebClient` se realiza mediante `onStatus`, que permite reaccionar a codigos de respuesta HTTP especificos antes de intentar deserializar el cuerpo:

```java
public Mono<UsuarioDTO> obtenerUsuarioSeguro(Long id) {
    return webClient.get()
        .uri("/usuarios/{id}", id)
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError, response -> {
            if (response.statusCode().value() == 404) {
                return Mono.error(new ResourceNotFoundException("Usuario no encontrado: " + id));
            }
            return response.bodyToMono(String.class)
                .flatMap(body -> Mono.error(new ClientException("Error cliente: " + body)));
        })
        .onStatus(HttpStatusCode::is5xxServerError, response ->
            Mono.error(new ServerException("Error en el servidor remoto")))
        .bodyToMono(UsuarioDTO.class);
}
```

El operador `exchangeToMono` ofrece un control mas granular sobre la respuesta completa, incluyendo headers y cuerpo raw:

```java
public Mono<UsuarioDTO> obtenerConFallback(Long id) {
    return webClient.get()
        .uri("/usuarios/{id}", id)
        .exchangeToMono(response -> {
            if (response.statusCode().is2xxSuccessful()) {
                return response.bodyToMono(UsuarioDTO.class);
            }
            return response.bodyToMono(ErrorResponse.class)
                .flatMap(err -> Mono.error(new ApiException(err.getMessage())));
        })
        .onErrorResume(ApiException.class, e -> {
            log.warn("Fallback para usuario {}: {}", id, e.getMessage());
            return Mono.just(UsuarioDTO.fallback(id));
        });
}
```

> Si no se maneja explicitamente, un error en el flujo se propagara como un `onError` signal. Si no hay ningun handler de errores en la cadena, la suscripcion fallara con una `reactor.core.Exceptions$ReactiveException`.

## Operador block()

El operador `block()` convierte un `Mono` o `Flux` reactivo en un resultado sincrono, suspendiendo la ejecucion hasta que el valor este disponible. Aunque esta disponible, su uso en el contexto de un servidor reactivo **debe evitarse** porque bloquea el hilo del reactor, anulando los beneficios del modelo no-bloqueante.

```java
// En un contexto reactivo - NO recomendado
UsuarioDTO usuario = webClient.get()
    .uri("/usuarios/1")
    .retrieve()
    .bodyToMono(UsuarioDTO.class)
    .block(); // Bloquea el hilo actual

// En un contexto sincrono (CLI, testing) - Aceptable
UsuarioDTO usuario = webClient.get()
    .uri("/usuarios/1")
    .retrieve()
    .bodyToMono(UsuarioDTO.class)
    .block(Duration.ofSeconds(5));
```

`block()` es util en escenarios de pruebas unitarias, scripts de linea de comandos o cuando se necesita interactuar con un API reactiva desde codigo sincrono. En el contexto de un endpoint reactivo, la respuesta correcta es retornar el `Mono` o `Flux` directamente para que el framework gestione la suscripcion.

