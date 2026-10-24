# File I/O con NIO.2

NIO.2, introducido en Java 7 como parte de JSR 203, representa una revision profunda del sistema de entrada/salida de Java. A diferencia de NIO original, que se centro en la E/S de red y los canales, NIO.2 aborda especificamente las operaciones con el sistema de archivos, proporcionando una API moderna, fluida y con soporte para operaciones mas alla de la simple lectura y escritura secuencial. El corazon de NIO.2 es la interfaz `Path` y la clase `Files`.

## Path: la ruta como objeto

`java.nio.file.Path` representa una ruta en el sistema de archivos. No es un String: es un objeto con metodos para navegar, resolver, relativizar y normalizar rutas. Se obtiene mediante el metodo factory `Path.of()` (Java 11+) o `Paths.get()` (Java 7+).

```java
Path home    = Path.of("/home", "usuario");
Path archivo = Path.of("/home/usuario/docs/./notas.txt");

Path normalizado = archivo.normalize();
// /home/usuario/docs/notas.txt

Path relativo = Path.of("docs/notas.txt");
Path absoluto  = relativo.toAbsolutePath();

Path docs = Path.of("/home/usuario/docs");
Path notas = docs.resolve("notas.txt");
// /home/usuario/docs/notas.txt

Path base  = Path.of("/home/usuario/docs");
Path target = Path.of("/home/usuario/docs/sub/archivo.txt");
Path relativePath = base.relativize(target);
// sub/archivo.txt
```

`resolve()` combina una ruta base con una ruta relativa. Si la ruta argumento es absoluta, retorna el argumento sin modificacion. `relativize()` construye la ruta relativa entre dos rutas. `normalize()` elimina elementos redundantes como `.` y `..`.

La interfaz `Path` tambien proporciona acceso a los componentes individuales de la ruta: `getFileName()`, `getParent()`, `getRoot()`, `getNameCount()` y `getName(int)`.

## Files: operaciones con archivos

La clase `java.nio.file.Files` contiene metodos estaticos para todas las operaciones comunes del sistema de archivos. Su diseno favorece el encadenamiento fluido y la devolucion de tipos adecuados.

**Lectura y escritura.** `readAllBytes(Path)` lee el contenido completo de un archivo en un arreglo de bytes. `readString(Path)` lo lee como String (Java 11+). `write(Path, byte[])` escribe un arreglo de bytes. `writeString(Path, CharSequence)` escribe un String.

```java
Path archivo = Path.of("datos.txt");

// Escritura
Files.writeString(archivo, "Contenido del archivo",
    StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);

// Lectura completa en memoria
String contenido = Files.readString(archivo, StandardCharsets.UTF_8);

// Lectura por lineas (eficiente para archivos grandes)
Files.lines(archivo).forEach(linea -> System.out.println(linea));
```

**Copia, movimiento y eliminacion.** `copy(Path, Path, CopyOption...)` copia un archivo. `move(Path, Path, CopyOption...)` lo mueve o renombra. `delete(Path)` lo elimina (lanza `NoSuchFileException` si no existe). `deleteIfExists(Path)` lo elimina silenciosamente si existe.

```java
Files.copy(Path.of("origen.txt"), Path.of("destino.txt"),
    StandardCopyOption.REPLACE_EXISTING);

Files.move(Path.of("temp.txt"), Path.of("final.txt"),
    StandardCopyOption.ATOMIC_MOVE);
```

**Creacion de directorios.** `createDirectory(Path)` crea un directorio (falla si el padre no existe). `createDirectories(Path)` crea todos los directorios intermedios necesarios.

```java
Files.createDirectories(Path.of("/tmp/a/b/c/d"));
```

**Existencia y atributos.** `exists(Path)` y `notExists(Path)` verifican la existencia. `isDirectory(Path)`, `isRegularFile(Path)`, `isReadable(Path)`, `isWritable(Path)` son metodos de consulta. `size(Path)` devuelve el tamano del archivo.

```java
if (Files.exists(Path.of("config.properties"))) {
    long tamano = Files.size(Path.of("config.properties"));
    System.out.println("Tamano: " + tamano);
}
```

## Listado y recorrido de directorios

`Files.list(Path)` devuelve un `Stream<Path>` con las entradas del directorio. `Files.walk(Path, int)` recorre el arbol recursivamente hasta la profundidad indicada. `Files.find(Path, int, BiPredicate)` recorre aplicando un filtro.

```java
// Listar archivos .java en un directorio
Files.list(Path.of("src"))
     .filter(p -> p.toString().endsWith(".java"))
     .forEach(System.out::println);

// Recorrer arbol recursivamente
Files.walk(Path.of("src"), 3)
     .filter(Files::isRegularFile)
     .forEach(System.out::println);
```

`Files.lines(Path)` devuelve un `Stream<String>` donde cada elemento es una linea del archivo, cerrando el recurso automaticamente al finalizar el stream. Es la forma mas eficiente de procesar archivos de texto grandes en Java moderno.

## BufferedReader y BufferedWriter con NIO.2

`Files.newBufferedReader(Path)` y `Files.newBufferedWriter(Path)` son equivalentes directos de `new BufferedReader(new FileReader(...))` pero aceptan `Path` y `Charset` explicitamente, evitando los problemas de codificacion por defecto de `FileReader`.

```java
try (BufferedReader reader = Files.newBufferedReader(
        Path.of("datos.txt"), StandardCharsets.UTF_8)) {
    String linea;
    while ((linea = reader.readLine()) != null) {
        System.out.println(linea);
    }
}
```

## Atributos de archivo

NIO.2 introduce un sistema extensible de atributos de archivo. `Files.readAttributes(Path, String, LinkOption...)` permite leer atributos de forma generica, y las interfaces del paquete `java.nio.file.attribute` proporcionan acceso tipado a atributos segun el sistema de archivos: `BasicFileAttributes`, `PosixFileAttributes`, `DosFileAttributes`.

```java
BasicFileAttributes attrs = Files.readAttributes(
    Path.of("archivo.txt"), BasicFileAttributes.class);

System.out.println("Creado: " + attrs.creationTime());
System.out.println("Modificado: " + attrs.lastModifiedTime());
System.out.println("Tamano: " + attrs.size());
System.out.println("Es directorio: " + attrs.isDirectory());
System.out.println("Es regular: " + attrs.isRegularFile());
```

Para sistemas POSIX, `PosixFileAttributes` anade permisos, propietario y grupo, permitiendo inspeccionar y modificar los permisos del archivo.

> A diferencia de `java.io.File`, que quedaba obsoleto en casi todos sus metodos, NIO.2 ofrece una API completa, bien disenada y con soporte nativo para enlaces simbolicos, atributos especificos de plataforma y operaciones atomicas. `java.io.File` debe considerarse obsoleto en favor de `Path` y `Files`.
