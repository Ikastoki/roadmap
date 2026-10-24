# Conexion a la base de datos

Establecer una conexion con la base de datos es el primer paso de cualquier operacion JDBC. La conexion representa un canal de comunicacion entre la aplicacion Java y el motor de base de datos, y sobre ella se ejecutan todas las sentencias SQL. En entornos modernos se prefiere el uso de `DataSource` (con pool de conexiones), pero el mecanismo basico mediante `DriverManager.getConnection()` sigue siendo valido para aplicaciones simples, pruebas unitarias y entornos de desarrollo.

## DriverManager.getConnection()

`DriverManager` es una clase del paquete `java.sql` que gestiona un conjunto de drivers JDBC registrados. Cuando se solicita una conexion, recorre los drivers registrados y delega en el primero que acepta la URL proporcionada. El metodo principal es:

```java
Connection conn = DriverManager.getConnection(String url, String user, String password);
```

Existen dos variantes adicionales:

```java
// Solo URL (usuario/contrasena en la propia URL)
Connection conn = DriverManager.getConnection(String url);

// Properties con parametros adicionales
Properties props = new Properties();
props.setProperty("user", "admin");
props.setProperty("password", "secret");
props.setProperty("ssl", "true");
Connection conn = DriverManager.getConnection(url, props);
```

En Java 6 y posteriores, los drivers se cargan automaticamente mediante el mecanismo ServiceLoader si el JAR del driver incluye el fichero `META-INF/services/java.sql.Driver`. En versiones anteriores era necesario registrar explicitamente el driver con `Class.forName("org.postgresql.Driver")`.

## URL de conexion segun base de datos

Cada motor de base de datos define su propio formato de URL JDBC, identificado por un prefijo unico que el DriverManager utiliza para seleccionar el driver adecuado.

### PostgreSQL

```java
String url = "jdbc:postgresql://localhost:5432/mi_bd";
String user = "postgres";
String password = "postgres";
Connection conn = DriverManager.getConnection(url, user, password);
```

Parametros adicionales comunes: `ssl=true`, `sslmode=require`, `ApplicationName=MiApp`, `connectTimeout=10` (segundos), `socketTimeout=30`. PostgreSQL utiliza el puerto 5432 por defecto.

### MySQL

```java
String url = "jdbc:mysql://localhost:3306/mi_bd";
String user = "root";
String password = "root";
Connection conn = DriverManager.getConnection(url, user, password);
```

MySQL requiere parametros adicionales para controlar la zona horaria (serverTimezone=UTC), el uso de SSL (useSSL=false en desarrollo) y la compatibilidad con el driver CJ (allowPublicKeyRetrieval=true en algunos casos). El puerto por defecto es 3306.

```java
String url = "jdbc:mysql://localhost:3306/mi_bd?serverTimezone=UTC&useSSL=false";
```

### H2

H2 es una base de datos embebida muy usada en desarrollo y pruebas. Soporta varios modos de conexion:

```java
// Modo embebido (archivo)
String url = "jdbc:h2:~/datos/mi_bd";  // ~ = directorio home del usuario

// Modo embebido en memoria
String url = "jdbc:h2:mem:testdb";

// Modo servidor (TCP)
String url = "jdbc:h2:tcp://localhost:9092/mi_bd";

Connection conn = DriverManager.getConnection(url, "sa", "");
```

### Oracle

```java
String url = "jdbc:oracle:thin:@localhost:1521:XE";
String user = "system";
String password = "oracle";
Connection conn = DriverManager.getConnection(url, user, password);
```

El formato es `jdbc:oracle:thin:@host:puerto:sid`. Para bases de datos con service name en lugar de SID se usa: `jdbc:oracle:thin:@//host:puerto/serviceName`.

## Propiedades de conexion

Ademas de usuario y contrasena, las conexiones JDBC aceptan propiedades adicionales que modifican el comportamiento de la sesion:

```java
Properties props = new Properties();
props.setProperty("user", "admin");
props.setProperty("password", "secreta");
props.setProperty("loginTimeout", "10");    // segundos
props.setProperty("connectTimeout", "10");
props.setProperty("socketTimeout", "30");
props.setProperty("tcpKeepAlive", "true");

Connection conn = DriverManager.getConnection(url, props);
```

Propiedades comunes:

| Propiedad | Descripcion |
|-----------|-------------|
| loginTimeout | Tiempo maximo de espera para establecer la conexion |
| connectTimeout | Timeout de conexion TCP (depende del driver) |
| socketTimeout | Timeout de lectura de socket |
| tcpKeepAlive | Mantiene vivo el socket TCP |
| ApplicationName | Identifica la aplicacion en los logs del servidor |
| ssl / useSSL | Habilita cifrado SSL/TLS |

## try-with-resources para gestion de conexiones

Desde Java 7, el bloque `try-with-resources` garantiza el cierre automatico de cualquier recurso que implemente `AutoCloseable`. `Connection`, `Statement` y `ResultSet` implementan esta interfaz, por lo que deben cerrarse siempre dentro de un try-with-resources:

```java
String url = "jdbc:postgresql://localhost:5432/mi_bd";
String user = "postgres";
String password = "postgres";

try (Connection conn = DriverManager.getConnection(url, user, password);
     PreparedStatement ps = conn.prepareStatement("SELECT 1");
     ResultSet rs = ps.executeQuery()) {

    while (rs.next()) {
        System.out.println(rs.getInt(1));
    }

} catch (SQLException e) {
    System.err.println("Error de conexion: " + e.getMessage());
}
```

El bloque try-with-resources cierra los recursos en orden inverso al de declaracion: primero `rs`, luego `ps`, finalmente `conn`. No es necesario un bloque `finally` explicito para el cierre. Si ocurre una excepcion durante el cierre (por ejemplo, una interrupcion de red), esta se suprime automaticamente y la excepcion original se propaga. Las excepciones suprimidas pueden recuperarse con `Throwable.getSuppressed()`.

> Es importante no cerrar la conexion dentro del mismo try-with-resources si se va a reutilizar en varias operaciones. En aplicaciones reales, la conexion se obtiene de un pool (HikariCP) y se devuelve al pool al cerrarla, no se destruye fisicamente. En esos casos, cada operacion debe crear su propio `Statement` y `ResultSet` dentro de try-with-resources, pero la conexion se gestiona desde el pool.
