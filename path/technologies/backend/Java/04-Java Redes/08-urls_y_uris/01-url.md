# URL (Uniform Resource Locator)

Una URL es una referencia a un recurso en la red que especifica tanto el mecanismo de acceso (protocolo) como la ubicacion fisica del recurso. En Java, la clase `java.net.URL` proporciona una abstraccion completa para trabajar con localizadores uniformes de recursos, permitiendo no solo analizar sus componentes sino tambien establecer conexiones directas con el servidor remoto.

## Construccion de una URL

La clase URL se instancia mediante su constructor, que acepta desde una cadena completa hasta sus componentes individuales. Java valida la URL en el momento de la creacion, lanzando `MalformedURLException` si el protocolo no es soportado o la sintaxis es incorrecta.

```java
URL url1 = new URL("https://api.example.com:443/path/to/resource?q=java&page=2#section");
URL url2 = new URL("https", "api.example.com", 443, "/path/to/resource?q=java");
```

## Componentes de una URL

Cada URL se descompone en seis partes fundamentales, accesibles mediante los metodos de instancia de la clase.

```java
URL url = new URL("https://user:pass@host.example.com:8080/path?query=val#ref");

System.out.println("Protocol: " + url.getProtocol());      // https
System.out.println("Host: " + url.getHost());              // host.example.com
System.out.println("Port: " + url.getPort());              // 8080
System.out.println("Default port: " + url.getDefaultPort());// 443
System.out.println("Path: " + url.getPath());              // /path
System.out.println("Query: " + url.getQuery());            // query=val
System.out.println("Ref: " + url.getRef());                // ref
System.out.println("Authority: " + url.getAuthority());    // user:pass@host.example.com:8080
System.out.println("User info: " + url.getUserInfo());     // user:pass
```

El metodo `getPort()` devuelve -1 si no se especifico puerto en la URL. En ese caso, `getDefaultPort()` entrega el puerto estandar del protocolo (80 para HTTP, 443 para HTTPS). `getHost()` puede devolver tanto un nombre de dominio como una direccion IP literal.

## Abrir conexiones desde una URL

El metodo `openConnection()` devuelve un objeto `URLConnection` especifico del protocolo (normalmente `HttpURLConnection` para HTTP/HTTPS). Permite configurar propiedades de la peticion antes de establecer la conexion real.

```java
URL url = new URL("https://jsonplaceholder.typicode.com/posts/1");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.setRequestProperty("Accept", "application/json");
int responseCode = conn.getResponseCode();
```

Para casos sencillos donde solo se necesita leer el contenido, `openStream()` es un atajo que abre directamente un `InputStream` sobre el recurso, combinando `openConnection().getInputStream()`.

```java
URL url = new URL("https://example.com/data.txt");
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(url.openStream()))) {
    reader.lines().forEach(System.out::println);
}
```

## Conversion a representacion textual

`toExternalForm()` devuelve la representacion completa de la URL como cadena, reconstruyendo todos sus componentes en el formato estandar. Es equivalente a llamar a `toString()`.

```java
URL url = new URL("https://example.com/path");
String external = url.toExternalForm(); // https://example.com/path
```

## Resolucion de URL relativas

La clase URL soporta resolucion de referencias relativas mediante el constructor que toma una URL base y una especificacion relativa. Esto es util al procesar documentos HTML que contienen enlaces relativos.

```java
URL base = new URL("https://example.com/docs/index.html");
URL rel = new URL(base, "images/logo.png");
System.out.println(rel.toExternalForm()); // https://example.com/docs/images/logo.png
```

El proceso de resolucion sigue las reglas de RFC 3986: si la URL relativa comienza con `/`, reemplaza la ruta completa desde la raiz del host; en caso contrario, resuelve contra el directorio de la URL base.

> Las URL en Java no soportan internacionalizacion directamente. Para dominios con caracteres no ASCII (IDN) debe usarse `java.net.IDN.toASCII()` antes de construir la URL, o utilizar `java.net.URI` que maneja la codificacion/decodificacion de forma mas completa.

## Comparacion y equals

Dos objetos URL se consideran iguales si y solo si apuntan al mismo recurso. La comparacion implica resolucion de nombres DNS, por lo que puede ser costosa y lanzar excepciones de red.

```java
URL a = new URL("http://example.com");
URL b = new URL("http://EXAMPLE.com");
System.out.println(a.equals(b)); // true (resuelve DNS)
```

Este comportamiento difiere de `URI.equals()`, que compara solo la representacion textual sin resolver la red.
