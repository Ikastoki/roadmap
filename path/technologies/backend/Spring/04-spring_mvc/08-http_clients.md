# HTTP Clients

Spring Framework proporciona múltiples clientes HTTP para consumir APIs externas, cada uno con características específicas para diferentes casos de uso. La elección del cliente depende de si se necesita sincronía o asincronía, si se requiere soporte reactivo, y si se prefiere una abstracción de alto nivel o control de bajo nivel sobre las peticiones HTTP.

`RestTemplate` es el cliente HTTP síncrono tradicional de Spring, ahora considerado legacy pero todavía ampliamente utilizado. Proporciona una API simple para realizar peticiones HTTP y deserializar automáticamente las respuestas. `RestTemplate` es ideal para aplicaciones que necesitan peticiones HTTP básicas sin complejidad adicional.

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(5)
            .errorHandler(new CustomErrorHandler())
            .build();
    }
}

@Service
public class UsuarioRestTemplateService {
    
    private final RestTemplate restTemplate;
    private final String apiUrl = "https://api.ejemplo.com/usuarios";
    
    public Usuario obtenerPorId(Long id) {
        return restTemplate.getForObject(apiUrl + "/{id}", Usuario.class, id);
    }
    
    public ResponseEntity<Usuario> obtenerEntity(Long id) {
        return restTemplate.getForEntity(apiUrl + "/{id}", Usuario.class, id);
    }
    
    public Usuario crear(Usuario usuario) {
        return restTemplate.postForObject(apiUrl, usuario, Usuario.class);
    }
    
    public void actualizar(Long id, Usuario usuario) {
        restTemplate.put(apiUrl + "/{id}", usuario, id);
    }
    
    public void eliminar(Long id) {
        restTemplate.delete(apiUrl + "/{id}", id);
    }
    
    public <T> ResponseEntity<T> exchange(String url, HttpMethod method, 
                                          HttpEntity<?> entity, 
                                          Class<T> responseType, 
                                          Object... uriVariables) {
        return restTemplate.exchange(url, method, entity, responseType, uriVariables);
    }
}
```

El manejo de errores en `RestTemplate` se realiza mediante `ErrorHandler`. Spring proporciona `DefaultResponseErrorHandler` que lanza `HttpClientErrorException` o `HttpServerErrorException` para respuestas 4xx y 5xx. Se puede crear un `ErrorHandler` personalizado para manejar errores de manera específica.

```java
public class CustomErrorHandler implements ResponseErrorHandler {
    
    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return response.getStatusCode().isError();
    }
    
    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        if (response.getStatusCode().is4xxClientError()) {
            // Manejar errores del cliente
            throw new ClientApiException("Error del cliente: " + 
                response.getStatusText());
        } else if (response.getStatusCode().is5xxServerError()) {
            // Manejar errores del servidor
            throw new ServerApiException("Error del servidor: " + 
                response.getStatusText());
        }
    }
}
```

`WebClient` es el cliente HTTP reactivo recomendado por Spring, introducido en Spring 5 como parte del ecosistema WebFlux. Proporciona una API funcional y no bloqueante que retorna `Mono` (para respuestas únicas) y `Flux` (para streams de datos). `WebClient` es ideal para aplicaciones que requieren alto rendimiento y no bloqueo.

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .baseUrl("https://api.ejemplo.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunctions
                .basicAuthentication("usuario", "password"))
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                    .responseTimeout(Duration.ofSeconds(5))
                    .observe(true)))
            .build();
    }
}

@Service
public class UsuarioWebClientService {
    
    private final WebClient webClient;
    
    public Mono<Usuario> obtenerPorId(Long id) {
        return webClient.get()
            .uri("/usuarios/{id}", id)
            .retrieve()
            .bodyToMono(Usuario.class)
            .timeout(Duration.ofSeconds(3))
            .onErrorResume(e -> Mono.error(new ApiException("Error obteniendo usuario")));
    }
    
    public Flux<Usuario> listarTodos() {
        return webClient.get()
            .uri("/usuarios")
            .retrieve()
            .bodyToFlux(Usuario.class);
    }
    
    public Mono<Usuario> crear(Usuario usuario) {
        return webClient.post()
            .uri("/usuarios")
            .bodyValue(usuario)
            .retrieve()
            .bodyToMono(Usuario.class);
    }
    
    public Mono<Usuario> actualizar(Long id, Usuario usuario) {
        return webClient.put()
            .uri("/usuarios/{id}", id)
            .bodyValue(usuario)
            .retrieve()
            .bodyToMono(Usuario.class);
    }
    
    public Mono<Void> eliminar(Long id) {
        return webClient.delete()
            .uri("/usuarios/{id}", id)
            .retrieve()
            .bodyToMono(Void.class);
    }
}
```

`BodyExtractors` de `WebClient` proporciona control granular sobre cómo se extraen los datos de la respuesta. Permite leer el body como un String completo, como un stream de bytes, o deserializarlo a objetos específicos. Esto es útil cuando se necesita procesar la respuesta de manera personalizada.

```java
public Mono<String> obtenerComoTexto(Long id) {
    return webClient.get()
        .uri("/usuarios/{id}", id)
        .retrieve()
        .bodyToMono(String.class);
}

public Mono<Map<String, Object>> obtenerComoMapa(Long id) {
    return webClient.get()
        .uri("/usuarios/{id}", id)
        .retrieve()
        .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {});
}

public Flux<String> obtenerStream() {
    return webClient.get()
        .uri("/events")
        .accept(MediaType.TEXT_EVENT_STREAM)
        .retrieve()
        .bodyToFlux(String.class);
}
```

El manejo de errores en `WebClient` se realiza mediante operators de Reactor como `onErrorResume`, `onErrorReturn` y `onErrorMap`. Estos operators permiten manejar diferentes tipos de errores de manera elegante, proporcionando respuestas por defecto, transformando errores o propagándolos según sea necesario.

```java
public Mono<Usuario> obtenerConManejoErrores(Long id) {
    return webClient.get()
        .uri("/usuarios/{id}", id)
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError, response -> 
            response.bodyToMono(String.class)
                .flatMap(body -> Mono.error(new ClientException(body))))
        .onStatus(HttpStatusCode::is5xxServerError, response -> 
            Mono.error(new ServerException("Error del servidor")))
        .bodyToMono(Usuario.class)
        .timeout(Duration.ofSeconds(3))
        .onErrorResume(TimeoutException.class, e -> 
            Mono.just(Usuario.fallback()))
        .onErrorResume(Exception.class, e -> 
            Mono.error(new ApiException("Error obteniendo usuario", e)));
}
```

`HttpClient` de Java 11+ es el cliente HTTP nativo de la plataforma Java. No es específico de Spring pero se puede integrar fácilmente con `WebClient` mediante `ReactorClientHttpConnector`. `HttpClient` de Java 11+ ofrece un rendimiento excelente y soporte para HTTP/2.

```java
@Bean
public WebClient webClient() {
    java.net.http.HttpClient httpClient = java.net.http.HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(5))
        .build();
    
    JdkClientHttpConnector connector = new JdkClientHttpConnector(httpClient);
    
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(connector))
        .build();
}
```

Los HTTP Interface Clients de Spring 6 son una abstracción declarativa para clientes HTTP, similar a los HTTP Clients de Feign. Permiten definir clientes como interfaces Java con anotaciones, donde Spring implementa automáticamente la lógica de llamada HTTP. Esta aproximación es altamente declarativa y reduce significativamente el código boilerplate.

```java
@HttpExchange(url = "/api/usuarios", accept = MediaType.APPLICATION_JSON_VALUE)
public interface UsuarioClient {
    
    @GetExchange("/{id}")
    Usuario obtenerPorId(@PathVariable Long id);
    
    @GetExchange
    List<Usuario> listar();
    
    @PostExchange
    Usuario crear(@RequestBody Usuario usuario);
    
    @PutExchange("/{id}")
    Usuario actualizar(@PathVariable Long id, @RequestBody Usuario usuario);
    
    @DeleteExchange("/{id}")
    void eliminar(@PathVariable Long id);
}

@Configuration
public class HttpClientConfig {
    
    @Bean
    public UsuarioClient usuarioClient(WebClient.Builder builder) {
        WebClient webClient = builder.baseUrl("https://api.ejemplo.com").build();
        return HttpServiceProxyFactory
            .forClientAdapter(WebClientAdapter.forClient(webClient))
            .createClient(UsuarioClient.class);
    }
}
```

> `WebClient` es la opción recomendada para nuevos proyectos, especialmente si se necesita alto rendimiento o integración con APIs reactivas. `RestTemplate` sigue siendo válido para casos simples y aplicaciones legacy, pero está en modo de mantenimiento y no recibirá nuevas funcionalidades.

