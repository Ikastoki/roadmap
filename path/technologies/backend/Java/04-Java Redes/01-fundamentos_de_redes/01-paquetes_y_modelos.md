# Paquetes y modelos de redes en Java

Java proporciona un ecosistema completo para la comunicacion en red que ha evolucionado durante tres decadas. Desde los bloques fundamentales en `java.net` hasta el cliente HTTP moderno en `java.net.http`, pasando por el subsistema de canales y buffers en `java.nio.channels`, la plataforma ofrece al desarrollador distintos niveles de abstraccion para abordar cualquier problema de comunicacion entre procesos remotos.

## Paquete java.net

El paquete `java.net` es el mas antiguo y contiene las clases clasicas para comunicacion mediante sockets. `Socket` y `ServerSocket` proporcionan comunicacion TCP orientada a conexion, mientras que `DatagramSocket` y `DatagramPacket` cubren UDP. `URL` y `URLConnection` permiten trabajar con recursos localizados por un localizador uniforme, y `URI` ofrece manipulacion de identificadores uniformes. `InetAddress` encapsula direcciones IP (IPv4 e IPv6) y ofrece resolucion de nombres DNS.

```java
URL url = new URL("https://api.ejemplo.com/datos");
URLConnection conn = url.openConnection();
BufferedReader reader = new BufferedReader(
    new InputStreamReader(conn.getInputStream()));
```

## Paquete java.nio.channels

Introducido en Java 1.4 como parte de New I/O (NIO), `java.nio.channels` proporciona un modelo de E/S no bloqueante basado en canales, buffers y selectores. `SocketChannel` y `ServerSocketChannel` son las contrapartidas no bloqueantes de `Socket` y `ServerSocket`. `Selector` permite multiplexar un unico hilo sobre multiples canales, eliminando la necesidad de un hilo por conexion.

```java
Selector selector = Selector.open();
SocketChannel canal = SocketChannel.open();
canal.configureBlocking(false);
canal.register(selector, SelectionKey.OP_READ);
```

Un canal es una abstraccion sobre un recurso de E/S que puede transferir datos directamente a un `ByteBuffer`. Los buffers soportan put/get de tipos primitivos, marcas, limites y capacidades.

## Paquete java.net.http

Desde Java 11, `java.net.http` proporciona `HttpClient`, `HttpRequest` y `HttpResponse` como el cliente HTTP moderno y preferido frente a `URLConnection`. Soporta HTTP/2, WebSocket, llamadas sincronas y asincronas con `CompletableFuture`.

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://api.ejemplo.com"))
    .GET()
    .build();
HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
```

## Paquete java.nio.file

NIO.2 (Java 7) anadio `java.nio.file` con `Path`, `Files` y `FileSystem`. Aunque orientado a sistema de archivos, su integracion con `FileChannel` permite operaciones de transferencia directa entre canales de red y disco mediante `transferTo()` y `transferFrom()`, evitando copias intermedias entre el espacio de usuario y el kernel.

## Modelos de E/S en red

Java soporta cuatro modelos fundamentales para la entrada/salida en red, cada uno con implicaciones directas en rendimiento, escalabilidad y complejidad de implementacion.

### Modelo bloqueante (BIO)

Es el modelo clasico de `java.net.Socket`. Cada llamada a `read()` o `write()` bloquea el hilo hasta que los datos esten disponibles o se haya completado la escritura. Es simple de programar pero no escala: se necesita un hilo por conexion, y cuando hay miles de conexiones inactivas el coste de context switching es prohibitivo.

### Modelo no bloqueante (NIO)

Con `SocketChannel` en modo no bloqueante, las llamadas retornan inmediatamente con los datos disponibles o indicando que no hay datos. El metodo `Selector.select()` permite a un unico hilo monitorizar cientos de miles de canales y reaccionar solo cuando hay datos listos para leer o espacio disponible para escribir.

```java
while (true) {
    selector.select(); // bloquea hasta que al menos un canal esta listo
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isReadable()) { /* leer datos */ }
        if (key.isWritable()) { /* escribir datos */ }
    }
}
```

### Modelo asincrono

`AsynchronousServerSocketChannel` y `AsynchronousSocketChannel` (Java 7) permiten iniciar operaciones de lectura o escritura que completan en segundo plano. El resultado se recibe mediante un `Future` o un `CompletionHandler`. El sistema operativo gestiona la E/S real y notifica al hilo cuando termina, desacoplando completamente el hilo de la operacion.

```java
AsynchronousSocketChannel canal = AsynchronousSocketChannel.open();
canal.connect(remoteAddress, null, new CompletionHandler<Void, Void>() {
    public void completed(Void result, Void attachment) { /* conectado */ }
    public void failed(Throwable exc, Void attachment) { /* error */ }
});
```

### Virtual Threads (Project Loom)

Desde Java 21, los virtual threads permiten escribir codigo bloqueante que escala como si fuera no bloqueante. Cada conexion puede ocupar un virtual thread ligero que solo consume recursos de pila mientras esta bloqueado en una operacion de E/S. El scheduler del JVM monta y desmonta virtual threads sobre hilos del sistema operativo (platform threads) segun sea necesario.

> Virtual threads no eliminan el bloqueo, sino que lo hacen economico. Un servidor puede albergar millones de virtual threads esperando en `accept()` o `read()` con un coste minimo de memoria y conmutacion. La API es la misma que BIO: `Socket`, `ServerSocket`, `InputStream`, `OutputStream`.

## Cuando usar cada modelo

La eleccion del modelo depende del perfil de carga. Para aplicaciones con pocas conexiones concurrentes (decenas o centenas), BIO con virtual threads es la opcion mas sencilla y productiva. Para sistemas que requieren el maximo rendimiento con miles de conexiones simultaneas y baja latencia, NIO con selectors ofrece control fino sobre el ciclo de eventos. El modelo asincrono es adecuado cuando se necesita integrar con APIs reactivas o cuando el flujo de trabajo implica operaciones de red intercaladas con computacion intensiva que no debe bloquearse mutuamente.
