# Bibliotecas principales

Java incluye un conjunto extenso de bibliotecas estandar que cubren desde operaciones fundamentales hasta networking avanzado. Estas bibliotecas forman el nucleo de la plataforma y se distribuyen junto con el JDK, eliminando la necesidad de dependencias externas para la mayoria de tareas comunes.

## java.lang

El paquete `java.lang` se importa automaticamente en todos los programas Java y contiene las clases fundamentales del lenguaje.

**String** representa secuencias inmutables de caracteres. Cada cadena创建 con comillas dobles se almacena en el String Pool, una zona especial del heap donde se reutilizan cadenas identicas para optimizar memoria. Los metodos mas utilizados son `length()`, `charAt()`, `substring()`, `indexOf()`, `replace()`, `split()`, `trim()` y `toLowerCase()`. La inmutabilidad implica que toda operacion sobre un String crea un nuevo objeto.

```java
String original = "  Hola Mundo  ";
String limpio = original.trim();          // "Hola Mundo"
String mayusculas = limpio.toUpperCase(); // "HOLA MUNDO"
boolean contiene = limpio.contains("Mundo"); // true
String[] partes = "a,b,c".split(",");    // ["a", "b", "c"]
```

**Math** proporciona metodos estaticos para operaciones matematicas: `abs()`, `max()`, `min()`, `sqrt()`, `pow()`, `random()`, `ceil()`, `floor()`. Tambien define constantes como `Math.PI` y `Math.E`. Todos sus metodos son estaticos y la clase no puede instanciarse.

**Object** es la raiz de la jerarquía de clases; todas las clases en Java extienden Object directa o indirectamente. Define los metodos fundamentales que todo objeto posee: `equals()`, `hashCode()`, `toString()`, `getClass()`, `clone()`, `finalize()` (deprecado desde Java 9) y los mecanismos de espera y notificacion para concurrencia: `wait()`, `notify()`, `notifyAll()`.

**System** ofrece acceso a funcionalidades del sistema operativo. `System.out` y `System.err` son los streams de salida y error estandar. `System.in` es el stream de entrada. `System.exit()` termina la JVM. `System.currentTimeMillis()` retorna el tiempo en milisegundos desde epoch. `System.getenv()` accede a variables de entorno del SO.

**Thread** representa un hilo de ejecucion. Cada programa Java tiene al menos un hilo (el hilo principal). La creacion de hilos se logra implementando `Runnable` o extendiendo `Thread`. Los metodos clave son `start()`, `run()`, `join()`, `sleep()`, `interrupt()` y `isAlive()`.

## java.util

`java.util` es una de las bibliotecas mas extensas del JDK, proporcionando estructuras de datos, utilidades y herramientas de fecha/hora.

**Collections Framework** define la arquitectura unificada para manejar grupos de objetos. Las interfaces principales son `Collection` (raiz de la jerarquía), `List` (colecciones ordenadas con duplicados), `Set` (colecciones sin duplicados) y `Map` (asociaciones clave-valor). Las implementaciones mas usadas son `ArrayList` (basado en arrays dinamicos), `LinkedList` (listas doblemente enlazadas), `HashSet` (basado en HashMap), `TreeSet` (ordenado naturalmente) y `HashMap` (tabla hash).

```java
import java.util.*;

List<String> nombres = new ArrayList<>();
nombres.add("Ana");
nombres.add("Carlos");
nombres.add("Ana");
// nombres contiene 3 elementos, ArrayList permite duplicados

Set<String> unicos = new HashSet<>(nombres);
// unicos contiene 2 elementos: "Ana" y "Carlos"

Map<String, Integer> edades = new HashMap<>();
edades.put("Ana", 25);
edades.put("Carlos", 30);
int edadAna = edades.get("Ana"); // 25
```

**Scanner** facilita la lectura de entrada desde multiples fuentes: `System.in`, archivos o cadenas. Se instancia con una fuente y utiliza metodos como `nextLine()`, `nextInt()`, `nextDouble()` para leer datos tipados. Es la forma mas comun de obtener entrada del usuario en programas de consola.

**Date y Calendar** son las clases originales de manejo de fechas, ambas con problemas de diseno conocidos: `Date` es mutable y mezcla conceptos de fecha y hora, mientras que `Calendar` es complejo y propenso a errores. Desde Java 8 se recomienda usar el paquete `java.time` que las reemplaza completamente.

## java.io

`java.io` proporciona un modelo de streams para operaciones de entrada y salida basado en decoradores (decorators). Los streams se clasifican por direccion (entrada/salida), contenido (bytes/caracteres) y comportamiento (flujo/buffer).

**InputStream/OutputStream** manejan datos en bruto (bytes). **Reader/Writer** manejan caracteres con soporte de codificacion. **BufferedReader/BufferedWriter** agregan buffering para mejorar rendimiento en operaciones de E/S.

```java
import java.io.*;

// Lectura de archivo con buffering
try (BufferedReader reader = new BufferedReader(new FileReader("datos.txt"))) {
    String linea;
    while ((linea = reader.readLine()) != null) {
        System.out.println(linea);
    }
} // el try-with-resources cierra automaticamente el reader

// Escritura de archivo
try (BufferedWriter writer = new BufferedWriter(new FileWriter("salida.txt"))) {
    writer.write("Primera linea");
    writer.newLine();
    writer.write("Segunda linea");
}
```

> Desde Java 7, el try-with-resources gestiona automaticamente el cierre de recursos que implementan `AutoCloseable`, eliminando la necesidad de bloques `finally` manuales para liberar streams.

## java.nio

`java.nio` (New I/O, introducido en Java 14 y expandido en Java 7 con NIO.2) ofrece una alternativa moderna al modelo de streams basada en canales y buffers. Es mas eficiente para operaciones de E/S no bloqueante y proporciona una API completa para manipulacion de archivos y directorios.

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;

// Leer un archivo completo
String contenido = Files.readString(Path.of("datos.txt"));

// Escribir un archivo
Files.writeString(Path.of("salida.txt"), "contenido", StandardCharsets.UTF_8);

// Listar directorios
try (DirectoryStream<Path> stream = Files.newDirectoryStream(Path.of("."))) {
    for (Path entry : stream) {
        System.out.println(entry.getFileName());
    }
}

// Copiar archivos
Files.copy(Path.of("origen.txt"), Path.of("destino.txt"),
           StandardCopyOption.REPLACE_EXISTING);
```

## java.sql

`java.sql` proporciona la API JDBC (Java Database Connectivity) para interactuar con bases de datos关系ales. Se basa en un modelo de interfaces que los fabricantes de drivers implementan. El flujo tipico es: establecer conexion, crear un statement, ejecutar la consulta, procesar el ResultSet y cerrar los recursos.

```java
import java.sql.*;

try (Connection conn = DriverManager.getConnection(
        "jdbc:postgresql://localhost:5432/mibase", "usuario", "password");
     PreparedStatement stmt = conn.prepareStatement(
        "SELECT nombre, email FROM usuarios WHERE edad > ?")) {

    stmt.setInt(1, 18);
    try (ResultSet rs = stmt.executeQuery()) {
        while (rs.next()) {
            String nombre = rs.getString("nombre");
            String email = rs.getString("email");
            System.out.println(nombre + ": " + email);
        }
    }
}
```

## java.net

`java.net` ofrece APIs para comunicacion de red. `URLConnection` y `HttpURLConnection` son las implementaciones base para peticiones HTTP. `Socket` y `ServerSocket` permiten comunicacion TCP de bajo nivel. Desde Java 11, `HttpClient` es la API moderna recomendada para peticiones HTTP asincronas con soporte para HTTP/2 y WebSockets.

```java
import java.net.http.*;
import java.net.URI;

HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.ejemplo.com/datos"))
    .header("Accept", "application/json")
    .GET()
    .build();

HttpResponse<String> response = client.send(request,
    HttpResponse.BodyHandlers.ofString());

System.out.println(response.statusCode());
System.out.println(response.body());
```

## java.time

`java.time` (Java 8+) es la API moderna de fecha y hora, immutable y thread-safe, que reemplaza a `Date` y `Calendar`. Se basa en el calendario ISO-8601 y ofrece una jerarquia clara de tipos: `LocalDate` (solo fecha), `LocalTime` (solo hora), `LocalDateTime` (fecha y hora sin zona), `ZonedDateTime` (fecha y hora con zona horaria), `Instant` (marcador de tiempo absoluto) y `Duration`/`Period` (cantidades de tiempo).

```java
import java.time.*;
import java.time.format.DateTimeFormatter;

LocalDate hoy = LocalDate.now();
LocalDate manana = hoy.plusDays(1);
DayOfWeek diaSemana = hoy.getDayOfWeek(); // LUNES, MARTES...

LocalDateTime momento = LocalDateTime.now();
String formateado = momento.format(
    DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm"));

Duration duracion = Duration.between(
    LocalTime.of(9, 0), LocalTime.of(17, 30));
long horas = duracion.toHours(); // 8
```
