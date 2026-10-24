# WebSockets en Java

El protocolo HTTP sigue un modelo request-response: el cliente inicia la comunicacion y el servidor responde. Para aplicaciones que requieren comunicacion bidireccional en tiempo real (chats, notificaciones, juegos, dashboards en vivo), este modelo resulta ineficiente. WebSocket (RFC 6455) establece un canal full-duplex persistente sobre una unica conexion TCP, permitiendo que servidor y cliente envien mensajes en cualquier momento sin la sobrecarga de las cabeceras HTTP en cada intercambio.

## Jakarta WebSocket API

Java estandariza el desarrollo de WebSocket mediante la especificacion Jakarta WebSocket (JSR 356), integrada en Jakarta EE y disponible para SE mediante implementaciones como Apache Tomcat, Eclipse Jetty o Tyrus.

### Dependencia Maven (Tomcat 10+)

```xml
<dependency>
    <groupId>jakarta.websocket</groupId>
    <artifactId>jakarta.websocket-api</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-core</artifactId>
    <version>11.0.0</version>
</dependency>
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-websocket</artifactId>
    <version>11.0.0</version>
</dependency>
```

## @ServerEndpoint

Un endpoint WebSocket en el servidor se define mediante la anotacion `@ServerEndpoint` con una URI de conexion. Los metodos del ciclo de vida se anotan con `@OnOpen`, `@OnMessage`, `@OnClose` y `@OnError`.

```java
import jakarta.websocket.*;
import jakarta.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

@ServerEndpoint("/chat")
public class ChatEndpoint {

    private static final Set<Session> sesiones
            = new CopyOnWriteArraySet<>();

    @OnOpen
    public void onOpen(Session session) {
        sesiones.add(session);
        broadcast("Usuario conectado. Total: " + sesiones.size());
    }

    @OnMessage
    public void onMessage(String mensaje, Session session)
            throws IOException {
        String nombre = session.getUserProperties()
                .getOrDefault("nombre", "Anonimo").toString();
        broadcast(nombre + ": " + mensaje);
    }

    @OnClose
    public void onClose(Session session) {
        sesiones.remove(session);
        broadcast("Usuario desconectado. Total: " + sesiones.size());
    }

    @OnError
    public void onError(Session session, Throwable error) {
        System.err.println("Error en sesion " + session.getId()
                + ": " + error.getMessage());
    }

    private void broadcast(String mensaje) {
        for (Session s : sesiones) {
            if (s.isOpen()) {
                try {
                    s.getBasicRemote().sendText(mensaje);
                } catch (IOException e) {
                    System.err.println(
                            "Error al enviar: " + e.getMessage());
                }
            }
        }
    }
}
```

### Explicacion

- `@ServerEndpoint("/chat")`: expone el endpoint en la URI `ws://servidor:puerto/chat`.
- `Session`: representa la conexion con un cliente. Permite enviar mensajes, acceder a parametros y almacenar propiedades de sesion (`getUserProperties()`).
- `CopyOnWriteArraySet`: conjunto seguro para entornos concurrentes, ideal para listas de sesiones con mas lecturas que escrituras.
- `broadcast()`: itera sobre todas las sesiones activas y envia el mensaje.

## Session y RemoteEndpoint

La interfaz `Session` proporciona metodos para interactuar con la conexion:

```java
session.getBasicRemote().sendText("texto");               // texto
session.getBasicRemote().sendObject(obj);                 // objeto (con encoder)
session.getBasicRemote().sendBinary(ByteBuffer.wrap(d));  // binario
session.getAsyncRemote().sendText("mensaje");             // asincrono
```

El `RemoteEndpoint.Basic` es sincrono (bloqueante); `RemoteEndpoint.Async` devuelve un `Future` para envio no bloqueante.

## Servidor embebido con Tomcat

Para ejecutar el endpoint sin un contenedor Jakarta EE completo:

```java
import org.apache.tomcat.websocket.server.WsServerContainer;
import jakarta.websocket.server.ServerContainer;
import org.apache.catalina.startup.Tomcat;
import java.io.File;

public class ServidorWebSocket {
    public static void main(String[] args) throws Exception {
        Tomcat tomcat = new Tomcat();
        tomcat.setPort(8080);
        tomcat.getConnector();

        // Registrar endpoint
        ServerContainer container = (ServerContainer)
            tomcat.getServletContext()
                    .addServlet("ws", "ws")
                    .getServletContext()
                    .getAttribute(
                "jakarta.websocket.server.ServerContainer");
        container.addEndpoint(ChatEndpoint.class);

        tomcat.start();
        System.out.println(
            "WebSocket en ws://localhost:8080/chat");
        tomcat.getServer().await();
    }
}
```

## Cliente WebSocket en Java

Jakarta WebSocket tambien proporciona una API cliente:

```java
import jakarta.websocket.*;

@ClientEndpoint
public class ChatCliente {

    @OnOpen
    public void onOpen(Session session) {
        System.out.println("Conectado al servidor");
    }

    @OnMessage
    public void onMessage(String mensaje) {
        System.out.println("Recibido: " + mensaje);
    }

    public static void main(String[] args) throws Exception {
        WebSocketContainer container =
                ContainerProvider.getWebSocketContainer();
        Session session = container.connectToServer(
                ChatCliente.class,
                new URI("ws://localhost:8080/chat"));
        session.getBasicRemote().sendText("Hola a todos!");
        Thread.sleep(5000);
        session.close();
    }
}
```

## Formato de los mensajes

Los mensajes WebSocket pueden ser de texto (String) o binarios (ByteBuffer). La eleccion depende del tipo de datos:

- **Texto**: JSON, XML, comandos de protocolo.
- **Binario**: imagenes, audio, datos comprimidos, protobuf.

```java
@OnMessage
public void onMessage(ByteBuffer datos, Session session) {
    // procesar mensaje binario
}

@OnMessage
public void onMessage(String texto, Session session) {
    // procesar mensaje de texto
}
```

> WebSocket no impone un formato de mensaje. Es responsabilidad de la aplicacion definir el protocolo. Lo mas comun es usar JSON como formato de texto, con una estructura que incluya tipo de mensaje y payload: `{"tipo": "mensaje", "datos": {...}}`.

## Diferencias con HTTP

| Caracteristica | HTTP | WebSocket |
|---|---|---|
| Modelo | Request-Response | Bidireccional |
| Conexion | Efimera | Persistente |
| Latencia | Alta (cabeceras) | Baja (sin cabeceras) |
| Overhead | Alto (cabeceras) | Minimo (2 bytes frame) |
| Cacheable | Si | No |
| Protocolo | Texto | Texto o binario |
