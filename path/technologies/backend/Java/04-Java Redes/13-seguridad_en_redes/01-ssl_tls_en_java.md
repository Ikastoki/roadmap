# SSL/TLS en Java

La capa de transporte segura (TLS) es el estandar actual para cifrar comunicaciones en redes IP. Su predecesor, SSL, esta completamente deprecado debido a vulnerabilidades como POODLE y CRIME. Java proporciona soporte nativo para TLS a traves del paquete `javax.net.ssl`, permitiendo establecer conexiones cifradas sin depender de bibliotecas externas.

## SSLContext

`SSLContext` es la fabrica central para toda la configuracion TLS. Se obtiene mediante el metodo de fabrica `getInstance()` especificando el protocolo.

```java
SSLContext contexto = SSLContext.getInstance("TLSv1.3");
```

Una vez creado, debe inicializarse con tres elementos:

- **KeyManager[]**: gestiona los certificados del servidor (quien firma, quien es el servidor).
- **TrustManager[]**: gestiona los certificados de confianza (a quien se le permite conectar).
- **SecureRandom**: fuente de entropia para la generacion de claves.

```java
contexto.init(keyManagers, trustManagers, new SecureRandom());
```

Para un cliente basico sin autenticacion mutua, los KeyManager pueden ser `null` y los TrustManager se obtienen del TrustStore por defecto.

> TLSv1.3 esta disponible desde Java 11 y ofrece mejoras significativas de seguridad y rendimiento frente a TLSv1.2: elimina modos inseguros (CBC, RC4), reduce el handshake a un solo round-trip (1-RTT) y elimina la negociacion de cifradores inseguros.

## SSLSocketFactory y SSLSocket

A partir de un `SSLContext` configurado se obtiene la fabrica de sockets seguros:

```java
SSLSocketFactory factory = contexto.getSocketFactory();
SSLSocket socket = (SSLSocket) factory.createSocket("ejemplo.com", 443);
```

`SSLSocket` extiende `Socket` y anade metodos para controlar parametros TLS:

```java
socket.setEnabledProtocols(new String[]{"TLSv1.3", "TLSv1.2"});
socket.setEnabledCipherSuites(new String[]{
    "TLS_AES_128_GCM_SHA256",
    "TLS_AES_256_GCM_SHA384"
});
```

El handshake TLS se inicia implicitamente al llamar a `getInputStream()` o `getOutputStream()`. Tambien se puede forzar con `socket.startHandshake()`.

## Cifradores (Cipher Suites)

Un cifrador es una combinacion de algoritmos de intercambio de claves, autenticacion, cifrado por bloques y MAC. La nomenclatura sigue el patron:

```
TLS_ALGORITMO_CIFRADO_MAC
```

Ejemplos comunes:

| Cipher Suite | Protocolo | Uso |
|---|---|---|
| TLS_AES_128_GCM_SHA256 | TLSv1.3 | Recomendado |
| TLS_AES_256_GCM_SHA384 | TLSv1.3 | Alta seguridad |
| TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 | TLSv1.2 | Amplio soporte |
| TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 | TLSv1.2 | Curvas elipticas |

```java
String[] cifradores = socket.getSupportedCipherSuites();
socket.setEnabledCipherSuites(cifradores);
```

> No deben habilitarse todos los cifradores soportados. Deben seleccionarse solo aquellos que ofrezcan **forward secrecy** (ECDHE, DHE) y cifrado autenticado (GCM, ChaCha20-Poly1305). Cifradores con CBC o RC4 deben desactivarse explicitamente.

## Certificados X.509 y almacenes de claves

Java gestiona los certificados mediante dos tipos de almacenes:

- **KeyStore** (`JKS` o `PKCS12`): contiene las claves privadas y sus certificados asociados. Se usa en el servidor para demostrar su identidad.
- **TrustStore**: contiene los certificados raiz de autoridades de certificacion (CA) en los que se confia. Se usa en el cliente para verificar la identidad del servidor.

```java
KeyStore keyStore = KeyStore.getInstance("PKCS12");
try (FileInputStream fis = new FileInputStream("keystore.p12")) {
    keyStore.load(fis, "password".toCharArray());
}

KeyManagerFactory kmf = KeyManagerFactory
        .getInstance(KeyManagerFactory.getDefaultAlgorithm());
kmf.init(keyStore, "password".toCharArray());
KeyManager[] kms = kmf.getKeyManagers();
```

Para el TrustStore:

```java
TrustManagerFactory tmf = TrustManagerFactory
        .getInstance(TrustManagerFactory.getDefaultAlgorithm());
tmf.init(keyStore); // o tmf.init((KeyStore) null) para el default
TrustManager[] tms = tmf.getTrustManagers();
```

## Ejemplo cliente HTTPS con SSLSocket

```java
SSLContext context = SSLContext.getInstance("TLSv1.3");
context.init(null, null, new SecureRandom());

SSLSocketFactory factory = context.getSocketFactory();
try (SSLSocket socket = (SSLSocket) factory.createSocket(
        "api.ejemplo.com", 443)) {

    socket.startHandshake();
    PrintWriter out = new PrintWriter(
            socket.getOutputStream(), true);
    BufferedReader in = new BufferedReader(
            new InputStreamReader(socket.getInputStream()));

    out.println("GET /datos HTTP/1.1");
    out.println("Host: api.ejemplo.com");
    out.println("Connection: close");
    out.println();

    String linea;
    while ((linea = in.readLine()) != null) {
        System.out.println(linea);
    }
}
```

> `SSLSocket` opera a nivel de socket; para comunicaciones HTTP seguras a nivel de aplicacion, se recomienda usar `HttpsURLConnection` o la API `HttpClient` introducida en Java 11, que abstrae la gestion del SSLContext.
