# HTTPS con HttpClient

La API `java.net.http.HttpClient`, introducida en Java 11 como parte del modulo `java.net.http`, es el sustituto moderno de `HttpURLConnection`. Soporta HTTP/2, envios sincronos y asincronos, y se integra de forma natural con `SSLContext` para conexiones HTTPS.

## Configuracion basica con SSLContext por defecto

Por defecto, `HttpClient` confia en las CA raiz del sistema operativo. Para conexiones HTTPS estandar no se requiere configuracion adicional:

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

HttpClient cliente = HttpClient.newHttpClient();

HttpRequest peticion = HttpRequest.newBuilder()
        .uri(URI.create("https://api.ejemplo.com/datos"))
        .GET()
        .build();

HttpResponse<String> respuesta = cliente.send(peticion,
        HttpResponse.BodyHandlers.ofString());

System.out.println(respuesta.body());
```

## SSLContext personalizado

Cuando se necesita un TrustStore especifico (por ejemplo, para confiar solo en un subconjunto de CA o en certificados internos), se construye un `SSLContext` y se pasa al builder del cliente:

```java
import javax.net.ssl.*;

// Cargar TrustStore personalizado
KeyStore trustStore = KeyStore.getInstance("PKCS12");
try (FileInputStream fis = new FileInputStream("truststore.p12")) {
    trustStore.load(fis, "cambio".toCharArray());
}

TrustManagerFactory tmf = TrustManagerFactory
        .getInstance(TrustManagerFactory.getDefaultAlgorithm());
tmf.init(trustStore);

SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
sslContext.init(null, tmf.getTrustManagers(), null);

// Construir cliente con SSLContext personalizado
HttpClient cliente = HttpClient.newBuilder()
        .sslContext(sslContext)
        .build();
```

## Certificados auto-firmados para desarrollo

En entornos de desarrollo es habitual usar certificados auto-firmados. Para que `HttpClient` los acepte, se necesita un `TrustManager` que no valide la cadena de confianza.

> Usar un TrustManager que confia en todo solo en entornos de desarrollo controlados. En produccion siempre debe validarse la cadena de certificados.

```java
TrustManager[] trustAllCerts = new TrustManager[]{
    new X509TrustManager() {
        public void checkClientTrusted(
                java.security.cert.X509Certificate[] c,
                String authType) {}

        public void checkServerTrusted(
                java.security.cert.X509Certificate[] c,
                String authType) {}

        public java.security.cert.X509Certificate[]
                getAcceptedIssuers() {
            return new java.security.cert.X509Certificate[0];
        }
    }
};

SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
sslContext.init(null, trustAllCerts, null);

HttpClient cliente = HttpClient.newBuilder()
        .sslContext(sslContext)
        .build();
```

## Configurar protocolos y cifradores

El `SSLContext` permite restringir los protocolos y cifradores habilitados:

```java
SSLParameters params = new SSLParameters();
params.setProtocols(new String[]{"TLSv1.3"});
params.setCipherSuites(new String[]{
    "TLS_AES_128_GCM_SHA256",
    "TLS_AES_256_GCM_SHA384"
});

HttpClient cliente = HttpClient.newBuilder()
        .sslContext(sslContext)
        .sslParameters(params)
        .build();
```

## Peticion HTTPS con autenticacion mutua (mTLS)

Para mTLS, donde el cliente tambien presenta un certificado, se configuran los `KeyManager` en el `SSLContext`:

```java
KeyStore keyStore = KeyStore.getInstance("PKCS12");
try (FileInputStream fis = new FileInputStream("cliente.p12")) {
    keyStore.load(fis, "password".toCharArray());
}

KeyManagerFactory kmf = KeyManagerFactory
        .getInstance(KeyManagerFactory.getDefaultAlgorithm());
kmf.init(keyStore, "password".toCharArray());

SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
sslContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);
```

## Peticion asincrona con SSL

`HttpClient` soporta operaciones asincronas mediante `CompletableFuture`:

```java
CompletableFuture<HttpResponse<String>> futuro =
    cliente.sendAsync(peticion,
            HttpResponse.BodyHandlers.ofString());

futuro.thenAccept(resp -> System.out.println(resp.body()));
```

La configuracion SSL es transparente tanto en modo sincrono como asincrono.

> La API HttpClient tambien soporta HTTP/2 por defecto y puede degradar a HTTP/1.1 si el servidor no lo soporta. La configuracion SSL es identica independientemente del protocolo utilizado.
