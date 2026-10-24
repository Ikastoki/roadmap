# Ejemplo completo: servidor echo

Un servidor echo es el equivalente al "Hola mundo" de la programacion de redes. El servidor recibe lineas de texto de los clientes y las devuelve intactas, precedidas por un prefijo indicativo. A pesar de su simplicidad, ilustra todos los elementos esenciales de una comunicacion TCP: establecimiento de conexion, lectura y escritura de flujos, manejo de concurrencia y cierre ordenado.

## Servidor Echo

El servidor utiliza `ServerSocket` para escuchar en el puerto 9090. Por cada conexion entrante, crea un nuevo hujo virtual (virtual thread) que ejecuta el manejador. El bucle principal nunca se bloquea porque delega la comunicacion al hilo recien creado.

```java
import java.io.*;
import java.net.*;
import java.util.concurrent.*;

public class EchoServer {

    public static void main(String[] args) {
        int port = 9090;

        try (ServerSocket serverSocket = new ServerSocket(port)) {
            System.out.println("Servidor Echo escuchando en puerto " + port);

            try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
                while (true) {
                    Socket clientSocket = serverSocket.accept();
                    executor.submit(() -> handleClient(clientSocket));
                }
            }

        } catch (IOException e) {
            System.err.println("Error en el servidor: " + e.getMessage());
        }
    }

    private static void handleClient(Socket clientSocket) {
        String clientAddress = clientSocket.getRemoteSocketAddress().toString();
        System.out.println("Cliente conectado: " + clientAddress);

        try (clientSocket;
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(clientSocket.getInputStream(), java.nio.charset.StandardCharsets.UTF_8));
             PrintWriter out = new PrintWriter(
                     new OutputStreamWriter(clientSocket.getOutputStream(),
                             java.nio.charset.StandardCharsets.UTF_8), true)) {

            String line;
            while ((line = in.readLine()) != null) {
                System.out.println("Recibido de " + clientAddress + ": " + line);
                out.println("Echo: " + line);
            }

            System.out.println("Cliente desconectado: " + clientAddress);

        } catch (IOException e) {
            System.err.println("Error con cliente " + clientAddress + ": " + e.getMessage());
        }
    }
}
```

La clase `Executors.newVirtualThreadPerTaskExecutor()` (Java 21+) crea un executor que lanza un nuevo virtual thread por cada tarea. Los virtual threads son lo suficientemente ligeros como para que no haya limite practico en el numero de conexiones concurrentes.

El bloque `try (clientSocket; ...)` incluye el `Socket` en el try-with-resources, por lo que se cierra automaticamente al salir del bloque. Esto envia la secuencia FIN al cliente.

## Cliente Echo

El cliente se conecta al servidor, lee lineas de la entrada estandar y las envia al servidor, mostrando la respuesta en la consola. Termina cuando el usuario escribe "salir".

```java
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;

public class EchoClient {

    public static void main(String[] args) {
        String host = "localhost";
        int port = 9090;

        try (Socket socket = new Socket(host, port);
             BufferedReader in = new BufferedReader(
                     new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
             PrintWriter out = new PrintWriter(
                     new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true);
             BufferedReader console = new BufferedReader(
                     new InputStreamReader(System.in, StandardCharsets.UTF_8))) {

            System.out.println("Conectado al servidor echo en " + host + ":" + port);
            System.out.println("Escribe mensajes (escribe 'salir' para terminar):");

            String userInput;
            while ((userInput = console.readLine()) != null) {
                if ("salir".equalsIgnoreCase(userInput.trim())) {
                    break;
                }

                out.println(userInput);
                String response = in.readLine();

                if (response == null) {
                    System.out.println("El servidor cerro la conexion.");
                    break;
                }

                System.out.println(response);
            }

        } catch (IOException e) {
            System.err.println("Error en el cliente: " + e.getMessage());
        }

        System.out.println("Cliente finalizado.");
    }
}
```

El cliente abre tres flujos: uno de entrada desde el teclado (`console`), otro de salida hacia el servidor (`out`) y otro de entrada desde el servidor (`in`). Cada linea escrita en consola se envia al servidor con `out.println()` y la respuesta se lee con `in.readLine()`.

> Es importante que tanto el servidor como el cliente usen la misma codificacion de caracteres. En este ejemplo ambos emplean `StandardCharsets.UTF_8`. Si uno usara ISO-8859-1 y otro UTF-8, los caracteres no ASCII se corromperian. El auto-flush en `PrintWriter` asegura que `println()` envie los datos inmediatamente, sin esperar a que el buffer se llene.

## Ejecucion

Para probar el sistema:

1. Compilar ambos archivos: `javac EchoServer.java EchoClient.java`
2. Iniciar el servidor: `java EchoServer`
3. En otra terminal, ejecutar el cliente: `java EchoClient`
4. Escribir mensajes en el cliente y observar las respuestas eco.

El servidor continua aceptando conexiones hasta que se termina el proceso con Ctrl+C. El cliente se desconecta correctamente al escribir "salir", lo que provoca que el servidor reciba `null` en `readLine()` y cierre el manejador.
