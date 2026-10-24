# NIO: Selectors (Multiplexores)

El selector es el componente que hace posible la E/S no bloqueante multiplexada. Sin el, el modo no bloqueante de los canales serviria de poco: el programador tendria que recorrer en bucle todos los canales preguntando si tienen datos disponibles, consumiendo CPU en espera activa (busy-waiting). El selector permite que un unico hilo monitoree cientos o miles de canales simultaneamente, durmiendo hasta que al menos uno de ellos este listo para una operacion de E/S.

## Selector

`java.nio.channels.Selector` actua como un multiplexor que se registra con uno o mas `SelectableChannel` (como `SocketChannel` y `ServerSocketChannel`) y notifica cuando alguno de ellos esta listo para ser leido, escrito, aceptado o conectado. Internamente, el selector delega en la llamada al sistema operativo `select()`, `poll()` o `epoll()` (dependiendo de la plataforma), lo que lo hace altamente eficiente.

```java
Selector selector = Selector.open();
```

El selector se crea mediante `Selector.open()` y se cierra con `close()`. Una vez cerrado, todas las claves registradas quedan invalidadas.

## RegistrationKey

Cuando un canal se registra en un selector, se devuelve un objeto `SelectionKey` que representa la relacion entre el canal y el selector. El metodo `register(Selector sel, int ops)` de `SelectableChannel` realiza el registro e indica las operaciones de interes mediante constantes de `SelectionKey`:

- `OP_ACCEPT` -- el canal (ServerSocketChannel) esta listo para aceptar una nueva conexion.
- `OP_CONNECT` -- el canal (SocketChannel) ha completado la conexion.
- `OP_READ` -- el canal tiene datos disponibles para leer.
- `OP_WRITE` -- el canal esta listo para recibir datos para escritura (tipicamente siempre esta listo, salvo que el buffer de salida del socket este lleno).

```java
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.bind(new InetSocketAddress(8080));

Selector selector = Selector.open();
SelectionKey key = serverChannel.register(selector, SelectionKey.OP_ACCEPT);
```

Se pueden combinar multiples operaciones con el operador OR binario: `SelectionKey.OP_READ | SelectionKey.OP_WRITE`.

## interestOps y readyOps

Cada `SelectionKey` mantiene dos conjuntos de operaciones:

**interestOps** define las operaciones que el canal esta interesado en monitorear. Puede modificarse en cualquier momento con `interestOps(int)`.

**readyOps** indica las operaciones que estan realmente listas en el momento de la ultima llamada a `select()`. Se consulta con `readyOps()` o mediante metodos booleanos:

- `isAcceptable()` -- equivalente a `(readyOps() & OP_ACCEPT) != 0`
- `isConnectable()` -- equivalente a `(readyOps() & OP_CONNECT) != 0`
- `isReadable()` -- equivalente a `(readyOps() & OP_READ) != 0`
- `isWritable()` -- equivalente a `(readyOps() & OP_WRITE) != 0`

## attach: datos asociados a la clave

El metodo `attach(Object ob)` de `SelectionKey` permite asociar cualquier objeto a la clave, como el estado de la conexion o un buffer para lectura. Es la forma habitual de mantener contexto por conexion sin recurrir a mapas externos. `attachment()` recupera el objeto asociado.

## Bucle principal del selector

El bucle tipico de un servidor NIO con selector sigue un patron fijo:

1. Llamar a `select()` para bloquearse hasta que al menos un canal este listo.
2. Obtener el conjunto de claves listas con `selectedKeys()`.
3. Iterar sobre las claves, procesando cada operacion lista.
4. Eliminar la clave procesada del conjunto de claves listas.

`select()` es bloqueante y retorna el numero de canales listos. Existen variantes: `select(long timeout)` que espera como maximo el tiempo indicado, y `selectNow()` que retorna inmediatamente sin bloquear (puede devolver 0).

```java
Selector selector = Selector.open();
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    int canalesListos = selector.select();

    if (canalesListos == 0) continue;

    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> iter = keys.iterator();

    while (iter.hasNext()) {
        SelectionKey key = iter.next();
        iter.remove();

        if (key.isAcceptable()) {
            ServerSocketChannel server = (ServerSocketChannel) key.channel();
            SocketChannel client = server.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);

        } else if (key.isReadable()) {
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(256);
            int bytesRead = client.read(buffer);

            if (bytesRead == -1) {
                client.close();
                continue;
            }

            buffer.flip();
            // Procesar datos recibidos
            buffer.clear();
        }
    }
}
```

## Ejemplo completo: servidor echo con selector

El siguiente ejemplo implementa un servidor echo que acepta multiples conexiones y reenvia los datos recibidos, todo en un unico hilo mediante un selector:

```java
public class EchoServer {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel server = ServerSocketChannel.open();
        server.configureBlocking(false);
        server.bind(new InetSocketAddress(8080));
        server.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            selector.select();
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();

            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();

                if (key.isAcceptable()) {
                    SocketChannel client = server.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);

                } else if (key.isReadable()) {
                    SocketChannel client = (SocketChannel) key.channel();
                    ByteBuffer buf = ByteBuffer.allocate(256);
                    int n = client.read(buf);

                    if (n == -1) {
                        client.close();
                        continue;
                    }

                    buf.flip();
                    client.write(buf);
                    buf.clear();
                }
            }
        }
    }
}
```

> El selector es la pieza fundamental que permite a Java competir con modelos de E/S como epoll en Linux o kqueue en BSD. Sin el, la E/S no bloqueante seria impracticable para aplicaciones con miles de conexiones simultaneas.
