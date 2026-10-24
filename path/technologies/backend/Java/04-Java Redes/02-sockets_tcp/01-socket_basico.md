# Socket basico

La comunicacion TCP en Java se construye sobre dos clases fundamentales: `Socket` para el extremo cliente y `ServerSocket` para el extremo servidor. TCP garantiza que los datos llegan en orden, sin perdidas y sin duplicados, estableciendo un canal bidireccional entre dos procesos que persiste durante toda la sesion.

## ServerSocket: el extremo servidor

`ServerSocket` escucha en un puerto especifico y acepta conexiones entrantes. El constructor vincula el socket a una direccion y puerto; si no se especifica direccion, se vincula a todas las interfaces disponibles.

```java
try (ServerSocket serverSocket = new ServerSocket(8080)) {
    System.out.println("Servidor escuchando en puerto 8080");
    while (true) {
        try (Socket cliente = serverSocket.accept()) {
            // Comunicacion con el cliente
        }
    }
}
```

El metodo `accept()` es bloqueante: el hilo se detiene hasta que un cliente establece conexion. Cuando esto ocurre, devuelve un objeto `Socket` que representa la conexion con ese cliente concreto, con su propia direccion y puerto remotos.

## Socket: el extremo cliente

El cliente crea un `Socket` especificando la direccion del servidor (como `String` o `InetAddress`) y el puerto. El constructor realiza la conexion TCP completa (three-way handshake) antes de retornar.

```java
try (Socket socket = new Socket("localhost", 8080)) {
    // Conexion establecida
    System.out.println("Conectado a " + socket.getRemoteSocketAddress());
}
```

## Flujos de datos

`Socket` proporciona dos metodos para obtener los flujos de entrada y salida: `getInputStream()` devuelve un `InputStream` del cual leer los datos que envia el otro extremo; `getOutputStream()` devuelve un `OutputStream` para enviar datos. Estos flujos operan a nivel de bytes.

### Flujos de bytes y caracteres

Para leer y escribir texto es habitual envolver los flujos base con readers y writers que manejan la conversion entre bytes y caracteres segun una codificacion:

```java
BufferedReader in = new BufferedReader(
    new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
PrintWriter out = new PrintWriter(
    new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true);
```

El segundo parametro `true` en `PrintWriter` activa el auto-flush, forzando el envio tras cada `println()`.

### Buffered streams

`BufferedInputStream` y `BufferedOutputStream` anaden un buffer interno que reduce las llamadas al sistema operativo. Cada lectura o escritura opera sobre una memoria intermedia (por defecto 8 KB), y el subyacente solo accede al sistema cuando el buffer se vacia o se llena.

```java
BufferedInputStream bis = new BufferedInputStream(socket.getInputStream());
BufferedOutputStream bos = new BufferedOutputStream(socket.getOutputStream());
```

### Data streams

`DataInputStream` y `DataOutputStream` permiten leer y escribir tipos primitivos de Java (int, long, double, boolean) en un formato binario portable entre plataformas. Son utiles cuando el protocolo intercambia valores numericos estructurados.

```java
DataOutputStream dos = new DataOutputStream(socket.getOutputStream());
dos.writeInt(42);
dos.writeUTF("Mensaje codificado");

DataInputStream dis = new DataInputStream(socket.getInputStream());
int valor = dis.readInt();
String texto = dis.readUTF();
```

### Object streams

`ObjectInputStream` y `ObjectOutputStream` serializan objetos Java completos a traves de la red. Requieren que las clases implementen `Serializable`. El flujo incluye metadatos de la clase, permitiendo reconstruir el objeto exacto en el receptor con sus campos.

```java
ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
oos.writeObject(new Mensaje("Hola", 1));

ObjectInputStream ois = new ObjectInputStream(socket.getInputStream());
Mensaje msg = (Mensaje) ois.readObject();
```

> La serializacion de objetos es comoda pero introduce overhead de rendimiento y vulnerabilidades de seguridad si se reciben objetos de fuentes no confiables. Para sistemas de produccion se recomienda usar formatos como JSON, Protocol Buffers o Apache Avro.

## Cierre con try-with-resources

Java 7 introdujo `try-with-resources`, que cierra automaticamente cualquier recurso que implemente `AutoCloseable`. `Socket`, `ServerSocket`, `InputStream` y `OutputStream` lo implementan. El cierre de un `Socket` tambien cierra sus flujos asociados.

```java
try (Socket socket = new Socket("localhost", 8080);
     BufferedReader in = new BufferedReader(
         new InputStreamReader(socket.getInputStream()));
     PrintWriter out = new PrintWriter(
         new OutputStreamWriter(socket.getOutputStream()), true)) {

    out.println("Hola servidor");
    String respuesta = in.readLine();
    System.out.println("Respuesta: " + respuesta);
}
```

El cierre ordenado de conexiones TCP sigue el protocolo: el extremo que cierra envia un segmento FIN, el receptor responde con ACK y posteriormente con su propio FIN. El metodo `close()` de `Socket` gestiona este intercambio, pero en situaciones de error puede enviar un RST en lugar de FIN.

## Ejemplo completo cliente-servidor

```java
// Servidor
public class ServidorBasico {
    public static void main(String[] args) throws IOException {
        try (ServerSocket ss = new ServerSocket(9090)) {
            System.out.println("Servidor en puerto 9090");
            try (Socket s = ss.accept();
                 BufferedReader in = new BufferedReader(
                     new InputStreamReader(s.getInputStream()));
                 PrintWriter out = new PrintWriter(
                     new OutputStreamWriter(s.getOutputStream()), true)) {

                String msg = in.readLine();
                out.println("Recibido: " + msg);
            }
        }
    }
}
```

```java
// Cliente
public class ClienteBasico {
    public static void main(String[] args) throws IOException {
        try (Socket s = new Socket("localhost", 9090);
             BufferedReader in = new BufferedReader(
                 new InputStreamReader(s.getInputStream()));
             PrintWriter out = new PrintWriter(
                 new OutputStreamWriter(s.getOutputStream()), true)) {

            out.println("Hola desde el cliente");
            System.out.println(in.readLine());
        }
    }
}
```
