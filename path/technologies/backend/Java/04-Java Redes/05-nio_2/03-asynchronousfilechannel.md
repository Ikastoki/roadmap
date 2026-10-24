# AsynchronousFileChannel

`AsynchronousFileChannel` forma parte del conjunto de canales asincronos introducidos en NIO.2. A diferencia de `FileChannel`, que requiere que el hilo invocante se bloquee hasta que la operacion de E/S se complete, `AsynchronousFileChannel` permite iniciar una operacion y recibir la notificacion del resultado en un momento posterior, ya sea mediante un `Future` o mediante un `CompletionHandler`. Esto libera al hilo para realizar otras tareas mientras el sistema operativo gestiona la transferencia de datos.

## Creacion del canal

Al igual que `FileChannel`, `AsynchronousFileChannel` se obtiene mediante metodos factory. El metodo mas comun es `open(Path, OpenOption...)`, que abre o crea un archivo y devuelve un canal asincrono.

```java
Path path = Path.of("datos.bin");
AsynchronousFileChannel channel = AsynchronousFileChannel.open(
    path, StandardOpenOption.READ, StandardOpenOption.WRITE,
    StandardOpenOption.CREATE);
```

Tambien existe una sobrecarga que acepta un `ExecutorService` para controlar el pool de hilos que procesara las operaciones de E/S. Si no se especifica, se crea un pool por defecto gestionado por el propio canal.

> `AsynchronousFileChannel` no soporta modo no bloqueante en el sentido de `SocketChannel` porque las operaciones de archivo siempre son inherentemente asincronas cuando se delegan a un pool de hilos. La diferencia no esta en el comportamiento del hilo, sino en quien notifica la finalizacion.

## Enfoque con Future

`read(ByteBuffer, long position)` y `write(ByteBuffer, long position)` devuelven un `Future<Integer>` que representa el resultado pendiente de la operacion. `position` indica la posicion en el archivo desde la que comenzar la operacion; los canales asincronos no mantienen una posicion interna, por lo que debe especificarse siempre.

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
Future<Integer> result = channel.read(buffer, 0);

// Mientras se lee, se puede hacer otro trabajo
System.out.println("Leyendo archivo...");

// Cuando se necesite el resultado, se bloquea si es necesario
int bytesRead = result.get();
System.out.println("Leidos: " + bytesRead + " bytes");

buffer.flip();
byte[] datos = new byte[buffer.remaining()];
buffer.get(datos);
System.out.println("Contenido: " + new String(datos, StandardCharsets.UTF_8));
```

El metodo `Future.get()` es bloqueante. Si se quiere saber si la operacion ha completado sin bloquearse, se usa `isDone()`. `Future.get(long timeout, TimeUnit unit)` permite esperar con timeout.

```java
Future<Integer> result = channel.read(buffer, 100);

while (!result.isDone()) {
    // Hacer otro trabajo mientras se espera
    System.out.println("Esperando lectura...");
}

int bytesRead = result.get();
```

## Enfoque con CompletionHandler

`CompletionHandler` es una interfaz con dos metodos: `completed(Integer result, ByteBuffer attachment)` que se invoca cuando la operacion tiene exito, y `failed(Throwable exc, ByteBuffer attachment)` que se invoca si ocurre un error. `attachment` es un objeto opcional que se pasa al handler y permite mantener contexto.

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

channel.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer bytesRead, ByteBuffer attachment) {
        attachment.flip();
        byte[] data = new byte[attachment.remaining()];
        attachment.get(data);
        System.out.println("Leidos: " + bytesRead + " bytes");
        System.out.println("Datos: " + new String(data, StandardCharsets.UTF_8));
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        System.err.println("Error de lectura: " + exc.getMessage());
    }
});

System.out.println("Lectura iniciada, el hilo principal continua...");
Thread.sleep(1000); // Esperar a que la lectura asincrona complete
```

La ventaja del enfoque con `CompletionHandler` sobre `Future` es que no requiere que ningun hilo se bloquee explicitamente esperando el resultado. El handler se invoca automaticamente cuando la operacion finaliza, en un hilo del pool del canal asincrono.

> Es importante tener en cuenta que el `CompletionHandler` se ejecuta en un hilo diferente al que inicio la operacion. Cualquier acceso a recursos compartidos debe estar debidamente sincronizado.

## Escritura asincrona

La escritura sigue el mismo patron que la lectura, utilizando `write(ByteBuffer, long position, A attachment, CompletionHandler)` o la variante con `Future`.

```java
String contenido = "Datos escritos de forma asincrona";
ByteBuffer buffer = ByteBuffer.wrap(contenido.getBytes(StandardCharsets.UTF_8));

channel.write(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        System.out.println("Escritos " + result + " bytes en el archivo");
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        System.err.println("Error de escritura: " + exc.getMessage());
    }
});
```

## Ejemplo completo: copia asincrona de archivos

El siguiente ejemplo copia el contenido de un archivo a otro utilizando `AsynchronousFileChannel` con `CompletionHandler`:

```java
public class AsyncFileCopy {
    public static void main(String[] args) throws Exception {
        Path origen  = Path.of("origen.bin");
        Path destino = Path.of("copia.bin");

        AsynchronousFileChannel inChannel = AsynchronousFileChannel.open(
            origen, StandardOpenOption.READ);
        AsynchronousFileChannel outChannel = AsynchronousFileChannel.open(
            destino, StandardOpenOption.WRITE, StandardOpenOption.CREATE,
            StandardOpenOption.TRUNCATE_EXISTING);

        ByteBuffer buffer = ByteBuffer.allocate(4096);
        long position = 0;

        inChannel.read(buffer, position, buffer, new CompletionHandler<>() {
            @Override
            public void completed(Integer bytesRead, ByteBuffer buf) {
                if (bytesRead == -1) {
                    try { inChannel.close(); outChannel.close(); } catch (IOException e) { }
                    System.out.println("Copia completada");
                    return;
                }

                buf.flip();
                outChannel.write(buf, position, buf, new CompletionHandler<>() {
                    @Override
                    public void completed(Integer result, ByteBuffer b) {
                        b.clear();
                        inChannel.read(b, position + result, b, this);
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer b) {
                        System.err.println("Error: " + exc.getMessage());
                    }
                });
            }

            @Override
            public void failed(Throwable exc, ByteBuffer buf) {
                System.err.println("Error: " + exc.getMessage());
            }
        });

        Thread.sleep(5000);
    }
}
```

## Consideraciones importantes

`AsynchronousFileChannel` no es adecuado para todos los escenarios. Para archivos pequenos (unos pocos KB), la sobrecarga de la capa asincrona supera cualquier beneficio potencial, y un `FileChannel` tradicional con un buffer directo ofrece mejor rendimiento. La E/S asincrona brilla cuando se trabaja con archivos grandes o cuando se realizan multiples operaciones de E/S concurrentes sobre diferentes archivos.

La posicion en las operaciones debe especificarse explicitamente porque `AsynchronousFileChannel` no mantiene una posicion interna. Esto facilita las operaciones concurrentes sobre el mismo canal: multiples lecturas y escrituras pueden ejecutarse simultaneamente en diferentes posiciones del mismo archivo.
