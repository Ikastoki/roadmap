# Autenticacion en APIs

Las APIs REST requieren un mecanismo para identificar al cliente que realiza la peticion y verificar que tiene permisos para acceder al recurso. HTTP define varios esquemas de autenticacion, cada uno con diferentes propiedades de seguridad, simplicidad y caso de uso. Los tres mas comunes en APIs modernas son Basic Auth, Bearer Token (JWT) y API Keys.

## Basic Auth

El esquema Basic transmite las credenciales (usuario y contrasena) en el encabezado `Authorization` codificadas en Base64. Es el mas simple de implementar pero tambien el menos seguro si no se combina con TLS.

```
Authorization: Basic base64(usuario:contrasena)
```

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.Base64;

public class BasicAuthEjemplo {
    public static void main(String[] args) throws Exception {
        String credenciales = "admin:secreto123";
        String encoded = Base64.getEncoder()
                .encodeToString(credenciales.getBytes());

        HttpClient cliente = HttpClient.newHttpClient();
        HttpRequest peticion = HttpRequest.newBuilder()
                .uri(URI.create("https://api.ejemplo.com/privado"))
                .header("Authorization", "Basic " + encoded)
                .GET()
                .build();

        HttpResponse<String> resp = cliente.send(peticion,
                HttpResponse.BodyHandlers.ofString());
        System.out.println(resp.body());
    }
}
```

> Basic Auth transmite las credenciales en cada peticion. **Nunca debe usarse sin TLS**, ya que Base64 es codificacion, no cifrado. Cualquier interceptor en la red puede decodificarlo y obtener las credenciales en texto plano.

## Bearer Token (JWT)

El esquema Bearer utiliza un token portador (bearer token) que el cliente envia en el encabezado `Authorization`. El mas extendido es JWT (JSON Web Token): un token con formato `cabecera.cuerpo.firma` que contiene claims (afirmaciones) sobre el usuario y esta firmado criptograficamente.

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0IiwibmFtZSI6IkFuYSJ9.abc123...
```

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class BearerTokenEjemplo {
    public static void main(String[] args) throws Exception {
        String token = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0In0.abc123";

        HttpClient cliente = HttpClient.newHttpClient();
        HttpRequest peticion = HttpRequest.newBuilder()
                .uri(URI.create("https://api.ejemplo.com/usuarios"))
                .header("Authorization", "Bearer " + token)
                .GET()
                .build();

        HttpResponse<String> resp = cliente.send(peticion,
                HttpResponse.BodyHandlers.ofString());
        System.out.println(resp.body());
    }
}
```

Para generar y verificar JWT en Java se usa la biblioteca `io.jsonwebtoken:jjwt`:

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;
import java.util.Date;

public class JwtEjemplo {
    public static void main(String[] args) {
        SecretKey clave = Keys
                .secretKeyFor(io.jsonwebtoken.SignatureAlgorithm.HS256);

        String token = Jwts.builder()
                .subject("1234")
                .claim("nombre", "Ana Garcia")
                .issuedAt(new Date())
                .expiration(new Date(
                        System.currentTimeMillis() + 3600_000))
                .signWith(clave)
                .compact();

        System.out.println("Token: " + token);

        // Verificar
        String sujeto = Jwts.parser()
                .verifyWith(clave)
                .build()
                .parseSignedClaims(token)
                .getPayload()
                .getSubject();
        System.out.println("Sujeto: " + sujeto);
    }
}
```

> El token JWT debe tener un tiempo de expiracion corto (minutos u horas). Si se necesita una sesion prolongada, se usa un refresh token almacenado de forma segura que permite obtener nuevos access tokens sin reautenticar al usuario.

## API Keys

Las API Keys son tokens opacos (tipicamente un string alfanumerico) que identifican a una aplicacion o cliente. Se envian en el encabezado personalizado `X-API-Key`.

```
X-API-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

```java
public class ApiKeyEjemplo {
    public static void main(String[] args) throws Exception {
        String apiKey = "a1b2c3d4-e5f6-7890-abcd-ef1234567890";

        HttpClient cliente = HttpClient.newHttpClient();
        HttpRequest peticion = HttpRequest.newBuilder()
                .uri(URI.create("https://api.ejemplo.com/datos"))
                .header("X-API-Key", apiKey)
                .GET()
                .build();

        HttpResponse<String> resp = cliente.send(peticion,
                HttpResponse.BodyHandlers.ofString());
        System.out.println(resp.body());
    }
}
```

Las API Keys son el metodo mas sencillo para autenticar maquinas (servicios, microservicios, scripts). A diferencia de Basic Auth, no requieren gestion de usuarios y contrasenas. Sin embargo, al ser opacas, no transportan informacion del usuario ni permisos granulares: o la clave es valida o no lo es.

| Esquema | Seguridad sin TLS | Estado | Rotacion | Caso de uso |
|---|---|---|---|---|
| Basic Auth | Nula | Sin estado | Compleja | Legacy, prototipos |
| Bearer JWT | Depende de la firma | Sin estado | Sencilla | APIs web, SPA |
| API Key | Baja | Sin estado | Sencilla | Servicio a servicio |

> En produccion, cualquier esquema de autenticacion debe combinarse con TLS. Un token JWT firmado sigue siendo vulnerable si se intercepta en texto plano, ya que un atacante puede reutilizarlo hasta que expire.
