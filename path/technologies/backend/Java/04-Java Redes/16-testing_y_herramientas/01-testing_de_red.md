# Testing de red en Java

Probar codigo que depende de la red presenta desafios que no aparecen en pruebas unitarias convencionales: la red es impredecible (latencias, timeouts, caidas), los servicios externos pueden no estar disponibles durante las pruebas y los efectos secundarios de enviar datos reales son inaceptables en entornos de integracion continua. Java dispone de varias estrategias y bibliotecas para aislar el codigo de red y hacer las pruebas deterministas.

## Mock de sockets con Mockito

La forma mas directa de probar codigo que usa `Socket` o `ServerSocket` es mediante mocking con Mockito, simulando las respuestas del otro extremo.

```java
import static org.mockito.Mockito.*;
import java.io.*;
import java.net.Socket;

public class ClienteRedTest {

    @Test
    public void testLecturaDeSocket() throws Exception {
        Socket socketMock = mock(Socket.class);
        InputStream input = new ByteArrayInputStream(
                "HOLA MUNDO\n".getBytes());
        when(socketMock.getInputStream()).thenReturn(input);

        // Pasar el mock al codigo bajo prueba
        ClienteRed cliente = new ClienteRed(socketMock);
        String resultado = cliente.leerLinea();
        assertEquals("HOLA MUNDO", resultado);
    }
}
```

Esta tecnica funciona bien para pruebas unitarias de clases que aceptan un `Socket` o `InputStream` por constructor, favoreciendo la inyeccion de dependencias.

## WireMock para servicios HTTP

WireMock simula servidores HTTP completos, permitiendo stub de respuestas, verificacion de peticiones y simulacion de fallos.

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock</artifactId>
    <version>3.9.0</version>
    <scope>test</scope>
</dependency>
```

```java
import static com.github.tomakehurst.wiremock.client.WireMock.*;
import com.github.tomakehurst.wiremock.junit5.WireMockTest;

@WireMockTest(httpPort = 8089)
public class ApiClientTest {

    @Test
    public void testGetUsuarios() throws Exception {
        stubFor(get(urlEqualTo("/api/usuarios"))
                .willReturn(aResponse()
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            [{"id":1,"nombre":"Ana"}]
                            """)
                        .withStatus(200)));

        HttpClient cliente = HttpClient.newHttpClient();
        HttpRequest req = HttpRequest.newBuilder()
                .uri(URI.create("http://localhost:8089/api/usuarios"))
                .GET()
                .build();

        HttpResponse<String> resp = cliente.send(req,
                HttpResponse.BodyHandlers.ofString());
        assertEquals(200, resp.statusCode());
        assertTrue(resp.body().contains("Ana"));
    }
}
```

WireMock tambien permite simular timeouts y errores:

```java
stubFor(get(urlPathMatching("/api/lento"))
        .willReturn(aResponse()
                .withFixedDelay(5000))); // 5 seg de retardo

stubFor(get(urlPathMatching("/api/error"))
        .willReturn(aResponse()
                .withStatus(503)
                .withBody("{\"error\":\"Servicio no disponible\"}")));
```

### Verificacion de peticiones

```java
verify(exactly(1), getRequestedFor(urlEqualTo("/api/usuarios")));
verify(exactly(0), postRequestedFor(anyUrl()));
```

> WireMock es la herramienta mas completa para simular APIs HTTP. Permite probar scenarios de integracion, tolerancia a fallos y comportamiento del cliente frente a respuestas inesperadas.

## MockWebServer (OkHttp)

MockWebServer es una alternativa ligera de Square (la misma compania que OkHttp) para simular servidores HTTP en pruebas.

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>mockwebserver</artifactId>
    <version>4.12.0</version>
    <scope>test</scope>
</dependency>
```

```java
import okhttp3.mockwebserver.*;
import okhttp3.*;
import org.junit.jupiter.api.*;

public class MockWebServerTest {

    private MockWebServer mockServer;

    @BeforeEach
    void setUp() throws Exception {
        mockServer = new MockWebServer();
        mockServer.start(8088);
    }

    @AfterEach
    void tearDown() throws Exception {
        mockServer.shutdown();
    }

    @Test
    void testPeticion() throws Exception {
        mockServer.enqueue(new MockResponse()
                .setResponseCode(200)
                .setHeader("Content-Type", "application/json")
                .setBody("{\"status\":\"ok\"}"));

        OkHttpClient client = new OkHttpClient();
        Request req = new Request.Builder()
                .url("http://localhost:8088/test")
                .build();

        Response resp = client.newCall(req).execute();
        assertEquals(200, resp.code());

        RecordedRequest peticion = mockServer.takeRequest();
        assertEquals("GET", peticion.getMethod());
        assertEquals("/test", peticion.getPath());
    }
}
```

## Test de timeouts

Probar que el codigo maneja correctamente los timeouts es fundamental. Tanto WireMock como MockWebServer permiten simular retardos:

```java
// WireMock
stubFor(get(anyUrl())
        .willReturn(aResponse()
                .withFixedDelay(10000)));

// MockWebServer
mockServer.enqueue(new MockResponse()
        .setBodyDelay(10, java.util.concurrent.TimeUnit.SECONDS));
```

```java
// Prueba de timeout en HttpClient
@Test
void testTimeout() {
    HttpClient cliente = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(2))
            .build();

    HttpRequest req = HttpRequest.newBuilder()
            .uri(URI.create("http://localhost:8088/lento"))
            .timeout(Duration.ofSeconds(1))
            .GET()
            .build();

    assertThrows(HttpTimeoutException.class, () ->
            cliente.send(req, HttpResponse.BodyHandlers.ofString()));
}
```

## Estrategias de testing

| Estrategia | Herramienta | Alcance |
|---|---|---|
| Mock unitario | Mockito | Clases que aceptan sockets/streams |
| Stub HTTP | WireMock, MockWebServer | Clientes HTTP |
| Pruebas de integracion | Testcontainers | Bases de datos, colas, servicios reales |
| Pruebas extremo a extremo | Docker Compose | Sistema completo en entorno aislado |

> Para testing de redes, la clave esta en aislar el codigo bajo prueba del sistema de red real. Las herramientas de stub permiten simular cualquier comportamiento (exito, error, latencia, caidas) de forma determinista y repetible.
