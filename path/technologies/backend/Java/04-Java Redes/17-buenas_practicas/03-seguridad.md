# Seguridad en redes

La seguridad en aplicaciones de red no es una caracteristica opcional ni una capa que se anade al final del desarrollo. Es una propiedad transversal que debe disenarse desde el primer dia. Una API vulnerable expone no solo los datos de la organizacion sino tambien los de sus usuarios, con consecuencias legales y reputacionales graves.

## Usar TLS/SSL siempre

Todo trafico de red en produccion debe viajar cifrado. No hay excusa para servir una API sin TLS, incluso en entornos internos donde la red se considera "confiable".

```java
// Configurar HttpsServer con certificado
import com.sun.net.httpserver.HttpsServer;
import javax.net.ssl.*;

HttpsServer server = HttpsServer.create(
        new InetSocketAddress(8443), 0);
SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
KeyManagerFactory kmf = KeyManagerFactory.getInstance(
        KeyManagerFactory.getDefaultAlgorithm());
KeyStore ks = KeyStore.getInstance("PKCS12");
ks.load(new FileInputStream("certificado.p12"),
        "password".toCharArray());
kmf.init(ks, "password".toCharArray());
sslContext.init(kmf.getKeyManagers(), null, null);
server.setHttpsConfigurator(new HttpsConfigurator(sslContext));
```

- Usar TLSv1.3 o TLSv1.2 como minimo. Deshabilitar SSLv3, TLSv1.0 y TLSv1.1.
- Los certificados deben ser emitidos por una CA reconocida, no auto-firmados, excepto en desarrollo.
- Renovar los certificados antes de que expiren; implementar alertas de vencimiento.

## Forzar HTTPS

El servidor debe redirigir automaticamente las peticiones HTTP a HTTPS:

```java
// Handler de redireccion
if (scheme.equals("http")) {
    exchange.getResponseHeaders().set("Location",
            "https://" + exchange.getRequestURI().toString());
    exchange.sendResponseHeaders(301, -1);
    return;
}
```

Y enviar la cabecera HSTS (HTTP Strict Transport Security) para que los navegadores solo se conecten via HTTPS en el futuro:

```java
exchange.getResponseHeaders().set(
        "Strict-Transport-Security",
        "max-age=31536000; includeSubDomains");
```

## Validar inputs

Toda entrada que llega por la red debe considerarse potencialmente maliciosa hasta que se demuestre lo contrario.

```java
// Validar longitud
if (nombre.length() > 100) {
    throw new ValidationException("Nombre demasiado largo");
}

// Validar formato
if (!email.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$")) {
    throw new ValidationException("Email invalido");
}

// Sanitizar para evitar XSS (si se muestra en HTML)
String sanitizado = Encode.forHtmlContent(input);
```

- Validar tamano maximo de cuerpos de peticion para evitar ataques de denial of service (DoS) por consumo de memoria.
- Validar tipos de contenido: si la API espera JSON, rechazar `Content-Type: application/xml`.
- Usar listas blancas (permitir solo lo conocido) en lugar de listas negras (bloquear lo malo conocido).

## Rate limiting

El rate limiting protege la API contra abusos, ataques de fuerza bruta y picos de trafico accidentales.

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class RateLimiter {
    private final ConcurrentHashMap<String, AtomicInteger>
            contadores = new ConcurrentHashMap<>();
    private final int maxPeticiones;
    private final long ventanaMs;

    public RateLimiter(int maxPeticiones, long ventanaMs) {
        this.maxPeticiones = maxPeticiones;
        this.ventanaMs = ventanaMs;
    }

    public boolean permitir(String clave) {
        contadores.putIfAbsent(clave, new AtomicInteger(0));
        int actual = contadores.get(clave).incrementAndGet();
        if (actual == 1) {
            // Resetear el contador tras la ventana
            new Thread(() -> {
                try { Thread.sleep(ventanaMs); }
                catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                contadores.remove(clave);
            }).start();
        }
        return actual <= maxPeticiones;
    }
}
```

```java
if (!rateLimiter.permitir(ipCliente)) {
    exchange.sendResponseHeaders(429, -1); // Too Many Requests
    return;
}
```

> Bucket4j y Resilience4j son bibliotecas maduras para rate limiting. En una API detras de un balanceador (Nginx, HAProxy), el rate limiting debe aplicarse tanto a nivel de balanceador como de aplicacion (defensa en profundidad).

## Headers de seguridad

Ademas de HSTS, las respuestas HTTP deben incluir:

```java
exchange.getResponseHeaders().set("X-Content-Type-Options", "nosniff");
exchange.getResponseHeaders().set("X-Frame-Options", "DENY");
exchange.getResponseHeaders().set("X-XSS-Protection", "0");
exchange.getResponseHeaders().set("Referrer-Policy", "no-referrer");
```

## Principio de minimo privilegio

- No exponer mas endpoints de los necesarios.
- No devolver mas datos de los necesarios en las respuestas.
- Usar diferentes niveles de autenticacion y autorizacion segun el recurso.
- No incluir informacion de la pila de tecnologia en mensajes de error: `"error interno"` en lugar de `"NullPointerException en linea 42 de UsuarioController.java"`.

> La seguridad no es un producto sino un proceso. Las vulnerabilidades aparecen y evolucionan; el codigo debe actualizarse periodicamente, las dependencias deben escanearse (OWASP Dependency-Check, Dependabot, Snyk) y deben realizarse auditorias de seguridad regulares.
