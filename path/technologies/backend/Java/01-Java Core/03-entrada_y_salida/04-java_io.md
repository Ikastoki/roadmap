# Java IO

La E/S (entrada/salida) en Java se organiza en torno a flujos (streams) que transportan datos byte a byte o carácter a carácter. La API original (`java.io`) define una arquitectura de clases basada en herencia y decoradores, mientras que la API NIO.2 (`java.nio.file`) introduce un modelo más moderno basado en rutas y utilidades estáticas. Ambas coexisten y se complementan en la práctica.

## Flujos de bytes vs caracteres

Java distingue dos familias de flujos: los de bytes (`InputStream`/`OutputStream`) y los de caracteres (`Reader`/`Writer`). Los primeros transportan datos binarios sin interpretación; los segundos decodifican y codifican caracteres según un charset.

`InputStream` y `OutputStream` son clases abstractas de las que derivan `FileInputStream`, `ByteArrayInputStream`, `SocketInputStream`, entre otras. `Reader` y `Writer` son su equivalente textual, con derivadas como `FileReader`, `BufferedReader` y `InputStreamReader`.

```java
import java.io.*;

public class EjemploFlujoBytes {
    public static void main(String[] args) {
        try (InputStream in = new FileInputStream("entrada.bin");
             OutputStream out = new FileOutputStream("salida.bin")) {
            byte[] buffer = new byte[1024];
            int leidos;
            while ((leidos = in.read(buffer)) != -1) {
                out.write(buffer, 0, leidos);
            }
        } catch (IOException e) {
            System.out.println("Error de E/S: " + e.getMessage());
        }
    }
}
```

> `read()` devuelve `-1` cuando no hay más datos disponibles. Este patrón de lectura en bucle es el estándar para copiar flujos de bytes.

## InputStreamReader y el puente entre bytes y caracteres

`InputStreamReader` es un adaptador que conecta un `InputStream` (bytes) con la interfaz `Reader` (caracteres). Es necesario cuando se necesita interpretar un flujo de bytes como texto con una codificación específica.

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

public class EjemploPuente {
    public static void main(String[] args) {
        try (Reader reader = new InputStreamReader(
                new FileInputStream("datos.txt"), StandardCharsets.UTF_8);
             BufferedReader br = new BufferedReader(reader)) {
            String linea;
            while ((linea = br.readLine()) != null) {
                System.out.println(linea);
            }
        } catch (IOException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

El uso de `StandardCharsets.UTF_8` evita la dependencia del charset por defecto del sistema, lo cual es esencial para portabilidad.

## BufferedReader y BufferedInputStream

Los flujos buffered envuelven otro flujo y añaden un buffer interno que reduce las operaciones de E/S al sistema operativo. Esto mejora significativamente el rendimiento cuando se leen muchos datos pequeños.

`BufferedReader` añade el método `readLine()` para leer línea a línea, y `BufferedInputStream` proporciona marca (`mark`) y reset (`reset`) para releer secciones del flujo.

```java
import java.io.*;

public class EjemploBuffered {
    public static void main(String[] args) {
        try (BufferedReader br = new BufferedReader(new FileReader("datos.txt"))) {
            StringBuilder sb = new StringBuilder();
            String linea;
            while ((linea = br.readLine()) != null) {
                sb.append(linea).append(System.lineSeparator());
            }
            System.out.println("Contenido completo:");
            System.out.println(sb);
        } catch (IOException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

> `StringBuilder` es preferible a la concatenación con `+` dentro de bucles, ya que cada `+` genera un nuevo objeto `String` intermedio. `StringBuilder` modifica un buffer mutable sin crear objetos adicionales.

## FileReader y FileWriter

`FileReader` y `FileWriter` son atajos que combinan `FileInputStream`/`FileOutputStream` con `InputStreamReader`/`OutputStreamWriter` usando el charset por defecto del sistema. Son útiles para E/S rápida de texto cuando no se necesita control de codificación.

```java
import java.io.*;

public class EjemploFileWriter {
    public static void main(String[] args) {
        try (FileWriter fw = new FileWriter("salida.txt");
             BufferedWriter bw = new BufferedWriter(fw)) {
            bw.write("Primera linea");
            bw.newLine();
            bw.write("Segunda linea");
            bw.newLine();
            bw.write("Tercera linea");
        } catch (IOException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

`FileWriter` en modo sobreescritura (por defecto) reemplaza el contenido del archivo. Para añadir al final se pasa `true` como segundo argumento: `new FileWriter("salida.txt", true)`.

## try-with-resources para auto-close

El bloque `try-with-resources` (Java 7+) garantiza que cualquier objeto que implemente `AutoCloseable` se cierre automáticamente al salir del bloque. Esto elimina la gestión manual de cierre y previene fugas de recursos.

```java
import java.io.*;

public class EjemploAutoClose {
    public static void copiarArchivo(String origen, String destino)
            throws IOException {
        try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(origen));
             BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(destino))) {
            byte[] buffer = new byte[8192];
            int bytesLeidos;
            while ((bytesLeidos = bis.read(buffer)) != -1) {
                bos.write(buffer, 0, bytesLeidos);
            }
        }
    }
}
```

> Cuando se declaran múltiples recursos en un solo `try`, se cierran en orden inverso al de declaración. Si alguno lanza una excepción al cerrarse, las excepciones de cierre posteriores se suprimen y se adjuntan como suppressed exceptions.

## Clase File

La clase `File` representa un camino del sistema de archivos. Proporciona métodos para consultar existencia, permisos, tamaño, y para crear o eliminar archivos y directorios. A pesar de que NIO.2 ofrece una alternativa más completa (`Path`), `File` sigue siendo ampliamente utilizado en código legado.

```java
import java.io.File;

public class EjemploFile {
    public static void main(String[] args) {
        File archivo = new File("datos.txt");

        System.out.println("Existe: " + archivo.exists());
        System.out.println("Es archivo: " + archivo.isFile());
        System.out.println("Es directorio: " + archivo.isDirectory());
        System.out.println("Tamanio: " + archivo.length() + " bytes");

        File nuevo = new File("nueva_carpeta");
        if (!nuevo.exists()) {
            nuevo.mkdir();
            System.out.println("Directorio creado.");
        }
    }
}
```

## NIO.2 Path y Files

Java 7 introdujo `java.nio.file.Path` y `java.nio.file.Files` como alternativa moderna a `File`. `Path` representa una ruta de sistema de archivos de forma más segura y portable, mientras `Files` proporciona métodos estáticos para operaciones comunes.

```java
import java.nio.file.*;
import java.io.IOException;
import java.util.List;

public class EjemploNIO2 {
    public static void main(String[] args) {
        Path ruta = Path.of("datos.txt");

        System.out.println("Existe: " + Files.exists(ruta));
        System.out.println("Nombre: " + ruta.getFileName());
        System.out.println("Absoluto: " + ruta.toAbsolutePath());

        try {
            List<String> lineas = Files.readAllLines(ruta);
            for (String linea : lineas) {
                System.out.println(linea);
            }

            Path copia = Path.of("datos_copia.txt");
            Files.copy(ruta, copia, StandardCopyOption.REPLACE_EXISTING);
            System.out.println("Copia creada: " + copia);

        } catch (IOException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

`Path.of()` es el método preferido para construir rutas. Admite segmentos variables con `Path.of("base", "subdir", "archivo.txt")`, lo que resulta portable entre sistemas operativos.

> `Files.readAllLines()` carga todo el archivo en memoria. Para archivos muy grandes se debe usar `Files.lines()` que devuelve un `Stream<String>` perezoso, o `Files.newBufferedReader()` para procesar línea a línea sin cargar todo en memoria.
