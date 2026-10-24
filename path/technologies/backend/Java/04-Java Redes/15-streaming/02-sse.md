# SSE (Server-Sent Events)

Server-Sent Events (SSE) es un estandar HTML5 que permite a un servidor enviar datos a un cliente web de forma unidireccional sobre una conexion HTTP persistente. A diferencia de WebSocket, SSE es mas simple (opera sobre HTTP estandar sin necesidad de upgrade de protocolo), no requiere bibliotecas adicionales en el cliente (navegadores lo soportan via `EventSource`) y maneja automaticamente la reconexion en caso de caida de la conexion.

> SSE es unidireccional: solo el servidor puede enviar datos. Si el cliente necesita enviar datos al servidor, debe hacerlo mediante peticiones HTTP independientes (POST). Para comunicacion bidireccional, WebSocket es la opcion adecuada.

## Formato text/event-stream

El servidor envia datos con el `Content-Type: text/event-stream`. Cada evento sigue el formato:

```
event: nombreEvento
data: contenido del mensaje
id: 123

```

Los campos `event` e `id` son opcionales. Cada evento se separa con una linea en blanco (doble salto de linea). El campo `data` puede tener multiples lineas; cada una se concatena con un salto de linea.

## Servidor SSE con HttpServer

```java
import com.sun.net.httpserver.*;
import java.io.*;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.Executors;

public class ServidorSSE {

    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(
                new InetSocketAddress(8080), 0);
        server.createContext("/eventos", new SSEHandler());
        server.setExecutor(Executors.newFixedThreadPool(4));
        server.start();
        System.out.println(
            "SSE en http://localhost:8080/eventos");
    }

    static class SSEHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange)
                throws IOException {
            if (!"GET".equalsIgnoreCase(
                    exchange.getRequestMethod())) {
                exchange.sendResponseHeaders(405, -1);
                return;
            }

            exchange.getResponseHeaders().set("Content-Type",
                    "text/event-stream; charset=UTF-8");
            exchange.getResponseHeaders().set("Cache-Control",
                    "no-cache");
            exchange.getResponseHeaders().set("Connection",
                    "keep-alive");
            exchange.sendResponseHeaders(200, 0); // chunked

            OutputStream out = exchange.getResponseBody();

            try {
                for (int i = 1; i <= 10; i++) {
                    String evento = "event: notificacion\n"
                            + "data: Evento numero " + i + "\n"
                            + "id: " + i + "\n\n";
                    out.write(evento.getBytes(
                            StandardCharsets.UTF_8));
                    out.flush();
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                out.close();
            }
        }
    }
}
```

## Eventos con JSON

El contenido de `data` suele ser JSON:

```java
String evento = "data: {\"usuario\":\"Ana\",\"accion\":\"login\"}\n\n";
out.write(evento.getBytes(StandardCharsets.UTF_8));
```

> En SSE, `data` puede tener multiples lineas concatenadas. Para enviar JSON con formato multilinea, cada linea debe ir prefijada con `data:`.

## Cliente HTML

El consumo de SSE en el navegador es nativo:

```html
<!DOCTYPE html>
<html>
<body>
    <ul id="lista"></ul>
    <script>
        const source = new EventSource('/eventos');

        source.addEventListener('notificacion', (e) => {
            const li = document.createElement('li');
            li.textContent = e.data + ' (id: ' + e.lastEventId + ')';
            document.getElementById('lista').appendChild(li);
        });

        source.onerror = () => console.log('Error SSE');
    </script>
</body>
</html>
```

## Reconexion automatica

SSE maneja la reconexion de forma nativa. Si la conexion se pierde, el cliente la restablece automaticamente enviando el ultimo `id` recibido en el encabezado `Last-Event-ID`. El servidor puede usar este ID para reenviar los eventos perdidos desde ese punto.

```java
String lastId = exchange.getRequestHeaders()
        .getFirst("Last-Event-ID");
// Reenviar eventos desde lastId...
```

El intervalo de reconexion se controla con el campo `retry` (en milisegundos):

```
retry: 3000

```

## Casos de uso

- Notificaciones push en tiempo real (alertas, mensajes).
- Dashboards con metricas actualizadas (stock, monitoreo).
- Feeds de actividad (redes sociales, logs).
- Actualizaciones de progreso de tareas largas.

> SSE no es adecuado para aplicaciones con alta frecuencia de mensajes (mas de varios cientos por segundo) o que requieran latencia de sub-milisegundo. Para esos casos, WebSocket o protocolos sobre TCP directo son mas apropiados.

## SSE vs WebSocket

| Caracteristica | SSE | WebSocket |
|---|---|---|
| Direccion | Servidor -> Cliente | Bidireccional |
| Protocolo | HTTP | ws:// wss:// |
| Reconexion | Automatica | Manual |
| Soporte navegador | Nativo (EventSource) | Nativo (WebSocket) |
| Latencia | Baja | Minimal |
| Max. conexiones por servidor | ~6 por dominio (HTTP/1.1) | Ilimitado |
| HTTP/2 | Limitado | Sin limite practico |
| Complejidad | Baja | Media |
