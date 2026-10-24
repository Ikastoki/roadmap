# File

La clase `java.io.File` proporciona una representacion abstracta de rutas de archivos y directorios en el sistema de archivos. No representa el contenido del archivo, sino su ubicacion y metadatos. Aunque funcional, su diseno presenta limitaciones que la API NIO.2 (`java.nio.file.Path`, `java.nio.file.Files`) viene a resolver.

## Creacion de instancias

```java
File f = new File("/home/usuario/documento.txt");
File directorio = new File("/home/usuario");
File relativo = new File("docs/archivo.md");
File desdePadre = new File("/home/usuario", "docs/archivo.md");
File desdeUri = new File(new URI("file:///home/usuario/doc.txt"));
```

La clase `File` es independiente de la plataforma: internamente normaliza las rutas usando el separador apropiado (`/` en Unix, `\` en Windows).

## Existencia y tipo

```java
boolean existe = f.exists();
boolean esArchivo = f.isFile();
boolean esDirectorio = f.isDirectory();
boolean esOculto = f.isHidden();
boolean esAbsoluto = f.isAbsolute();
boolean sePuedeLeer = f.canRead();
boolean sePuedeEscribir = f.canWrite();
boolean sePuedeEjecutar = f.canExecute();
```

## Informacion de la ruta

```java
String nombre = f.getName();            // "documento.txt"
String ruta = f.getPath();              // "home/usuario/documento.txt"
String rutaAbs = f.getAbsolutePath();   // "/home/usuario/documento.txt"
String canonical = f.getCanonicalPath();// resuelve .. y enlaces simbolicos
String padre = f.getParent();           // "/home/usuario"
```

> `getCanonicalPath()` lanza `IOException` porque accede al sistema de archivos para resolver enlaces simbolicos. `getAbsolutePath()` no lanza excepciones pero puede contener elementos como `..` o `.`.

## Metadatos

```java
long tamanio = f.length();              // bytes
long ultimaMod = f.lastModified();      // milisegundos desde epoch
boolean setReadOnly = f.setReadOnly();
boolean setLastMod = f.setLastModified(System.currentTimeMillis());
```

## Operaciones con directorios

```java
boolean creado = directorio.mkdir();    // crea el directorio (falla si padre no existe)
boolean creados = directorio.mkdirs();  // crea directorios intermedios necesarios
```

Listado de contenido:

```java
File[] archivos = directorio.listFiles();
String[] nombres = directorio.list();
File[] filtrados = directorio.listFiles((dir, name) -> name.endsWith(".txt"));
```

`listFiles()` acepta un `FileFilter` o `FilenameFilter` para filtrar resultados. Retorna `null` si el `File` no es un directorio o si ocurre un error de E/S.

## Operaciones con archivos

```java
boolean renombrado = f.renameTo(new File("/otra/ruta/doc.txt"));
boolean eliminado = f.delete();         // elimina archivo o directorio vacio
f.deleteOnExit();                       // elimina al terminar la JVM
```

```java
File temp = File.createTempFile("prefijo", ".tmp");
temp.deleteOnExit();
```

`createTempFile()` crea un archivo temporal con nombre unico en el directorio temporal del sistema.

## Limitaciones de java.io.File

- No puede leer ni escribir contenido del archivo (requiere `FileInputStream`, `FileOutputStream`, `FileReader`, `FileWriter`).
- Los metodos `listFiles()` y `length()` retornan sin garantias en sistemas de archivos con cambios concurrentes.
- Manejo deficiente de enlaces simbolicos.
- Falta de soporte para operaciones masivas (copiar, mover, eliminar arboles).
- No proporciona notificacion de cambios en el sistema de archivos.
- Los metodos `delete()` y `renameTo()` no lanzan excepciones; retornan boolean que puede ser insuficiente para diagnosticar errores.

## Alternativa NIO.2: Path y Files

`java.nio.file.Path` y `java.nio.file.Files` (Java 7+) ofrecen una API mas solida y funcional.

```java
Path path = Paths.get("/home/usuario/documento.txt");
Path desdeUri = Paths.get(URI.create("file:///home/usuario/doc.txt"));
Path rel = Paths.get("docs", "archivo.md"); // usa separador correcto
```

Metodos de `Path`:

```java
Path absoluto = path.toAbsolutePath();
Path normalizado = path.normalize();    // resuelve .. y .
Path resuelto = path.resolve("subdir"); // /home/usuario/documento.txt/subdir
Path padre = path.getParent();
Path nombre = path.getFileName();
int niveles = path.getNameCount();
Path subpath = path.subpath(0, 2);      // [home, usuario]
```

Operaciones con `Files`:

```java
boolean existe = Files.exists(path);
boolean esDir = Files.isDirectory(path);
long tamanio = Files.size(path);

// Crear
Path creado = Files.createFile(path);
Path dirs = Files.createDirectories(Paths.get("a/b/c"));

// Copiar, mover, eliminar
Files.copy(origen, destino, StandardCopyOption.REPLACE_EXISTING);
Files.move(origen, destino);
Files.delete(path);           // lanza NoSuchFileException
Files.deleteIfExists(path);   // retorna boolean

// Leer y escribir
List<String> lineas = Files.readAllLines(path);
byte[] bytes = Files.readAllBytes(path);
Files.write(path, "contenido".getBytes());
```

```java
// Recorrido recursivo
Files.walk(Paths.get("/home/usuario"))
     .filter(p -> p.toString().endsWith(".txt"))
     .forEach(System.out::println);
```

```java
// Stream con filtro
Files.list(directorioPath)
     .filter(Files::isRegularFile)
     .collect(Collectors.toList());
```

## File vs Path: resumen

`File` es adecuado para codigo legacy y operaciones simples. `Path`/`Files` es preferible para codigo nuevo por:

- Excepciones claras (`NoSuchFileException`, `FileAlreadyExistsException`).
- Operaciones atomicas con opciones (`COPY_ATTRIBUTES`, `REPLACE_EXISTING`, `ATOMIC_MOVE`).
- Soporte para enlaces simbolicos (`isSymbolicLink()`, `readSymbolicLink()`).
- Visitas recursivas con `FileVisitor` y `walk()`.
- Atributos de archivo especificos por sistema de archivos (`BasicFileAttributes`, `PosixFileAttributes`).
- Notificacion de cambios con `WatchService`.
