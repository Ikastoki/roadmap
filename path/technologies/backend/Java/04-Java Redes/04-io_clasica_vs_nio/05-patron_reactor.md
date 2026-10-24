# Patron Reactor en detalle

El patron Reactor es un patron de diseno de concurrencia especificamente concebido para manejar E/S en sistemas con alta demanda de conexiones simultaneas. Su objetivo es demultiplexar las solicitudes entrantes y despacharlas a los manejadores correspondientes utilizando un unico hilo o un numero reducido de ellos, evitando la creacion de un hilo por conexion. Este patron es la base arquitectonica de frameworks como Netty, Reactor y, en buena medida, del propio NIO de Java.

## Componentes del patron Reactor

El patron Reactor se compone de cuatro elementos esenciales que colaboran para lograr la demultiplexacion y el despacho eficiente de eventos de E/S:

**Reactor (Demultiplexor de eventos).** Es el nucleo del patron. Mantiene un bucle de eventos que invoca la llamada al sistema `select()` (o `epoll()`, `kqueue()`) para determinar que canales tienen datos disponibles o estan listos para operaciones de E/S. Cuando detecta un evento, lo despacha al manejador correspondiente. En terminos de NIO, el Reactor envuelve un `Selector`.

**Acceptor.** Es un manejador especializado que se encarga de aceptar nuevas conexiones entrantes. Cuando el Reactor detecta un evento de `OP_ACCEPT` en el `ServerSocketChannel`, delega en el Acceptor, que crea un nuevo `SocketChannel`, lo configura en modo no bloqueante, crea un `Handler` para esa conexion y lo registra en el Reactor para eventos de lectura.

**Handler.** Es el componente que procesa los datos de una conexion especifica. Implementa la logica de negocio para leer, interpretar y responder a las solicitudes del cliente. Cada conexion activa tiene su propio Handler asociado, y este Handler puede mantener estado local (buffers parciales, contexto de la sesion).

**Demultiplexer (Sincrono).** Es la llamada al sistema operativo que permite al Reactor determinar que canales estan listos. En Java NIO, el `Selector` abstrae esta funcionalidad y oculta las diferencias entre plataformas.

## Flujo de operacion

El flujo tipico de un servidor basado en Reactor es el siguiente:

1. El Reactor inicia y registra un `ServerSocketChannel` con `OP_ACCEPT`.
2. El Reactor entra en su bucle y llama a `select()`, bloqueandose hasta que haya eventos.
3. Cuando llega una nueva conexion, el Reactor detecta `OP_ACCEPT` y llama al Acceptor.
4. El Acceptor acepta la conexion, crea un `SocketChannel`, lo configura como no bloqueante y crea un Handler asociado.
5. El Handler se registra en el Reactor para `OP_READ`.
6. El Reactor continua su bucle. Cuando el cliente envie datos, detectara `OP_READ` en ese canal y llamara al Handler correspondiente.
7. El Handler lee los datos, los procesa y puede registrar `OP_WRITE` si necesita enviar una respuesta.
8. Cuando el cliente cierra la conexion, el Handler detecta el fin del stream y cierra el canal.

## Ejemplo: servidor Reactor en Java

El siguiente codigo implementa un servidor HTTP minimo basado en el patron Reactor con NIO. Aunque simplificado, muestra la estructura esencial del patron.

```java
public class Reactor implements Runnable {
    final Selector selector;
    final ServerSocketChannel serverChannel;

    Reactor(int port) throws IOException {
        selector = Selector.open();
        serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress(port));
        serverChannel.register(selector, SelectionKey.OP_ACCEPT, new Acceptor());
    }

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                selector.select();
                Set<SelectionKey> selected = selector.selectedKeys();
                Iterator<SelectionKey> it = selected.iterator();
                while (it.hasNext()) {
                    dispatch((SelectionKey) it.next());
                    it.remove();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    void dispatch(SelectionKey k) {
        Runnable handler = (Runnable) k.attachment();
        if (handler != null) {
            handler.run();
        }
    }

    class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                SocketChannel clientChannel = serverChannel.accept();
                if (clientChannel != null) {
                    new HttpHandler(selector, clientChannel);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

El Handler para cada conexion:

```java
public class HttpHandler implements Runnable {
    final SocketChannel channel;
    final SelectionKey key;
    final ByteBuffer buffer = ByteBuffer.allocate(1024);

    HttpHandler(Selector selector, SocketChannel channel) throws IOException {
        this.channel = channel;
        this.channel.configureBlocking(false);
        this.key = channel.register(selector, 0);
        this.key.attach(this);
        this.key.interestOps(SelectionKey.OP_READ);
        selector.wakeup();
    }

    @Override
    public void run() {
        try {
            if (key.isReadable()) {
                read();
            } else if (key.isWritable()) {
                write();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    void read() throws IOException {
        int n = channel.read(buffer);
        if (n == -1) {
            channel.close();
            return;
        }
        buffer.flip();
        String request = StandardCharsets.UTF_8.decode(buffer).toString();
        System.out.println("Request: " + request);
        buffer.clear();
        key.interestOps(SelectionKey.OP_WRITE);
    }

    void write() throws IOException {
        String response = "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello, World!";
        ByteBuffer resp = ByteBuffer.wrap(response.getBytes(StandardCharsets.UTF_8));
        channel.write(resp);
        channel.close();
    }
}
```

> El metodo `selector.wakeup()` en el constructor de `HttpHandler` es necesario porque el Reactor puede estar bloqueado en `select()` cuando se registra un nuevo canal. `wakeup()` hace que `select()` retorne inmediatamente, permitiendo que el bucle procese la nueva clave registrada.

## Variantes del patron

Existen dos variantes principales del patron Reactor:

**Reactor de un solo hilo.** Todo el procesamiento (deteccion de eventos, lectura, escritura y logica de negocio) ocurre en un unico hilo. Es simple pero no aprovecha multiples nucleos. Adecuado para cargas de trabajo fundamentalmente limitadas por E/S.

**Reactor con pool de hilos.** El Reactor detecta los eventos pero delega la logica de negocio a un pool de hilos de trabajo. Los handlers de lectura y escritura siguen ejecutandose en el hilo del Reactor, pero el procesamiento de la solicitud se realiza en paralelo. Es la arquitectura que utilizan frameworks como Netty.
