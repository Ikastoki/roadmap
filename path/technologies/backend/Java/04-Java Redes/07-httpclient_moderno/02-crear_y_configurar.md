# Crear y configurar HttpClient

La creacion de un `HttpClient` sigue el patron builder, lo que permite encadenar configuraciones antes de construir la instancia final. Una vez construido, el cliente es inmutable y seguro para su uso desde multiples hilos. Esto significa que todas las opciones de configuracion deben establecerse durante la construccion.

## HttpClient.newBuilder()

El punto de entrada es `HttpClient.newBuilder()`, que devuelve un `HttpClient.Builder`. Sobre este builder se llaman los metodos de configuracion y finalmente `build()` produce la instancia lista para usar. Existe tambien `HttpClient.newHttpClient()` como atajo que devuelve un cliente con valores por defecto.

```java
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(15))
        .followRedirects(HttpClient.Redirect.NORMAL)
        .build();
```

## Configuracion de version de protocolo

El metodo `version()` acepta `HttpClient.Version.HTTP_2` o `HttpClient.Version.HTTP_1_1`. Por defecto, el cliente intenta HTTP/2. Si se fuerza `HTTP_1_1`, todas las peticiones usaran HTTP/1.1 sin negociacion ALPN.

```java
// Forzar HTTP/1.1
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_1_1)
        .build();
```

> Forzar HTTP/1.1 puede ser util cuando se trabaja con servidores legacy, proxies corporativos o balanceadores que no manejan correctamente la negociacion ALPN. En caso de duda, mantener HTTP_2 permite degradacion automatica.

## connectTimeout

El timeout de conexion limita el tiempo maximo de espera para establecer la conexion TCP con el servidor. Se configura con un `java.time.Duration`. Si transcurre el tiempo sin establecer conexion, se lanza `java.net.http.HttpConnectTimeoutException`.

```java
HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(5))
        .build();
```

> `connectTimeout` solo controla la fase de establecimiento de conexion, no la lectura de la respuesta. Para limitar el tiempo total de una peticion se debe usar `java.util.concurrent.CompletableFuture.orTimeout()` en llamadas asincronas o `HttpRequest.timeout()`.

## followRedirects

Controla la politica de redireccion automatica. Acepta tres valores del enum `HttpClient.Redirect`:

- `NEVER`: no se siguen redirecciones. El desarrollador recibe la respuesta 3xx y debe manejarla manualmente.
- `ALWAYS`: se siguen todas las redirecciones, incluso entre protocolos (de HTTP a HTTPS).
- `NORMAL` (recomendado): se siguen redirecciones pero solo si no cambia el protocolo (de HTTP a HTTP, de HTTPS a HTTPS). Es el valor por defecto.

```java
HttpClient client = HttpClient.newBuilder()
        .followRedirects(HttpClient.Redirect.ALWAYS)
        .build();
```

## executor

Por defecto, `sendAsync` ejecuta las tareas en el *common fork-join pool* (`ForkJoinPool.commonPool()`). Para aplicaciones que requieren aislamiento de recursos, control sobre el numero de hilos o integracion con un pool gestionado, se puede proporcionar un `Executor` personalizado mediante el metodo `executor()`.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

HttpClient client = HttpClient.newBuilder()
        .executor(executor)
        .build();

// Importante: cerrar el executor cuando ya no se necesite
executor.shutdown();
```

> Si no se especifica un `Executor`, el cliente comparte el pool comun con otros componentes del JVM, lo que puede provocar contencion en sistemas con muchas peticiones concurrentes. Para aplicaciones en produccion es recomendable usar un executor dedicado.

## sslContext

Para conexiones HTTPS que requieren certificados de cliente, confianzas personalizadas o protocolos TLS especificos, se utiliza `sslContext()`. Recibe un `javax.net.ssl.SSLContext` configurado con los KeyManager, TrustManager o algoritmos deseados.

```java
SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
sslContext.init(null, null, null); // TrustManager por defecto

HttpClient client = HttpClient.newBuilder()
        .sslContext(sslContext)
        .build();
```

Tambien existe `sslParameters()` para ajustar parametros TLS como los protocolos habilitados o los cipher suites, sin necesidad de crear un SSLContext completo.

```java
SSLParameters params = new SSLParameters();
params.setProtocols(new String[] {"TLSv1.3", "TLSv1.2"});

HttpClient client = HttpClient.newBuilder()
        .sslParameters(params)
        .build();
```

## proxy

`HttpClient` soporta proxies HTTP mediante `ProxySelector`. Se puede configurar un proxy fijo o un selector personalizado.

```java
// Proxy fijo
HttpClient client = HttpClient.newBuilder()
        .proxy(ProxySelector.of(new InetSocketAddress("proxy.example.com", 8080)))
        .build();

// Sin proxy (conexion directa)
HttpClient client = HttpClient.newBuilder()
        .proxy(ProxySelector.of(null))
        .build();
```

Si no se especifica `proxy()`, el cliente usa `ProxySelector.getDefault()`, que lee las propiedades del sistema `http.proxyHost`, `http.proxyPort`, etc.

```java
// Configuracion completa
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(10))
        .followRedirects(HttpClient.Redirect.NORMAL)
        .executor(Executors.newCachedThreadPool())
        .sslContext(sslContext)
        .proxy(ProxySelector.getDefault())
        .build();
```
