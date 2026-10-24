# URI (Uniform Resource Identifier)

Un URI es una secuencia de caracteres que identifica un recurso de manera univoca, ya sea por su ubicacion (URL), por su nombre (URN) o por ambos. La clase `java.net.URI` representa URIs conforme a RFC 2396 y posteriores, y sirve como base estructural para la mayoria de operaciones de identificacion de recursos en Java, incluyendo aquellas que requieren codificacion/decodificacion de caracteres especiales.

## Creacion de un URI

La clase URI ofrece varios constructores que permiten especificar desde una cadena completa hasta cada componente por separado. A diferencia de URL, URI no requiere validacion de red ni soporte del protocolo en el momento de la creacion.

```java
URI uri1 = new URI("https://user@host.example.com:8080/path/to/file?query=val#frag");
URI uri2 = new URI("https", "user@host.example.com", "/path/file", "query=val", "frag");
URI uri3 = new URI("https", "host.example.com", "/path/file", "query");
URI uri4 = new URI("urn:isbn:0-486-27557-4");
```

El constructor por cadena lanza `URISyntaxException` si la sintaxis no es valida, mientras que los constructores con componentes individuales lanzan `IllegalArgumentException` cuando un componente es inapropiado.

## Componentes de un URI

Los metodos de acceso son similares a los de URL, pero con diferencias sutiles. URI distingue entre componentes interpretados (raw) y decodificados.

```java
URI uri = new URI("https://user@host.com:8080/path/file.txt?q=hello%20world#sec1");

System.out.println("getScheme(): " + uri.getScheme());          // https
System.out.println("getAuthority(): " + uri.getAuthority());    // user@host.com:8080
System.out.println("getUserInfo(): " + uri.getUserInfo());      // user
System.out.println("getHost(): " + uri.getHost());              // host.com
System.out.println("getPort(): " + uri.getPort());              // 8080
System.out.println("getPath(): " + uri.getPath());              // /path/file.txt
System.out.println("getQuery(): " + uri.getQuery());            // q=hello world
System.out.println("getFragment(): " + uri.getFragment());      // sec1
System.out.println("getRawQuery(): " + uri.getRawQuery());      // q=hello%20world
```

Los metodos `getRaw*()` devuelven los componentes sin decodificar, preservando la codificacion percentual original. Esto es critico cuando se necesita reconstruir el URI exactamente como fue recibido.

## Diferencia entre URL y URI

Toda URL es un URI, pero no todo URI es una URL. La diferencia fundamental radica en que una URL ademas de identificar proporciona el mecanismo de acceso (protocolo), mientras que un URI puede ser simplemente un nombre abstracto como un URN.

```java
URI urn = new URI("urn:isbn:0-486-27557-4");
System.out.println(urn.isAbsolute()); // true
// new URL("urn:isbn:0-486-27557-4") -> MalformedURLException: unknown protocol
```

| Aspecto | URL | URI |
|---------|-----|-----|
| Identifica recurso | Si | Si |
| Proporciona acceso | Si | No |
| Requiere soporte protocolo | Si | No |
| equals resuelve DNS | Si | Solo comparacion textual |
| Permite URN | No | Si |

El metodo `URI.toURL()` convierte un URI absoluto con esquema conocido en un objeto URL, lanzando `IllegalArgumentException` si el esquema no es soportado por el gestor de URLs.

## Resolucion y relativizacion

Dos de las operaciones mas potentes de URI son `resolve()` y `relativize()`. La primera permite obtener un URI absoluto a partir de uno relativo contra una base; la segunda realiza la operacion inversa.

```java
URI base = new URI("https://example.com/docs/index.html");
URI relative = new URI("images/logo.png");
URI resolved = base.resolve(relative);
System.out.println(resolved); // https://example.com/docs/images/logo.png

URI full = new URI("https://example.com/docs/images/logo.png");
URI relativized = base.relativize(full);
System.out.println(relativized); // images/logo.png
```

## Normalizacion

El metodo `normalize()` elimina componentes redundantes de la ruta como `.` y `..`, produciendo un URI semanticamente equivalente pero mas limpio.

```java
URI raw = new URI("https://example.com/a/./b/../c/file.txt");
URI normalized = raw.normalize();
System.out.println(normalized); // https://example.com/a/c/file.txt
```

## Comparacion

`URI.equals()` compara dos URIs basandose exclusivamente en sus componentes, sin resolver DNS. Esto lo hace mas predecible y eficiente que `URL.equals()`.

```java
URI a = new URI("http://EXAMPLE.com/path");
URI b = new URI("http://example.com/path");
System.out.println(a.equals(b)); // false (difiere en host)
```

Para comparacion sin distincion de mayusculas debe normalizarse el host previamente. `compareTo()` sigue el orden lexicografico definido en la especificacion.

## isAbsolute

Un URI es absoluto si tiene un esquema (protocolo). Los URIs relativos carecen de esquema y deben resolverse contra una base para ser utilizados.

```java
URI abs = new URI("https://example.com");
URI rel = new URI("/path/to/file");
System.out.println(abs.isAbsolute()); // true
System.out.println(rel.isAbsolute()); // false
```

> A diferencia de URL, URI permite representar recursos sin determinar como acceder a ellos. Esta separacion entre identificacion y localizacion es clave en arquitecturas de servicios web, donde un mismo recurso puede ser accesible por multiples protocolos.
