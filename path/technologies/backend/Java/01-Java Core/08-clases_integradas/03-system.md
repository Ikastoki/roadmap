# System

La clase `java.lang.System` es una clase de utilidad de sistema que proporciona acceso a recursos del entorno de ejecucion. No puede instanciarse (constructor privado) y todos sus miembros son `static`. Actua como fachada ante el sistema operativo subyacente, la JVM y el entorno del usuario.

## Entrada y salida estandar

```java
System.out.println("salida estandar");
System.err.println("salida de error");
int dato = System.in.read(); // byte desde entrada estandar
```

`System.out` y `System.err` son instancias de `PrintStream` configuradas para escribir en la salida y error estandar del sistema operativo. `System.in` es un `InputStream` que lee de la entrada estandar.

Pueden redirigirse:

```java
System.setOut(new PrintStream("salida.txt"));
System.setErr(new PrintStream("error.log"));
System.setIn(new FileInputStream("entrada.txt"));
```

## Terminacion de la JVM

```java
System.exit(0);   // terminacion normal
System.exit(1);   // terminacion con error
```

`System.exit(int status)` termina la JVM invocando el metodo `halt()` de `Runtime`. El argumento es el codigo de retorno para el sistema operativo (0 indica exito, distinto de cero indica error). Antes de terminar, ejecuta los hooks de apagado registrados con `Runtime.addShutdownHook()`.

> `System.exit()` nunca retorna. No debe usarse en aplicaciones web o servidores; es preferible lanzar una excepcion o permitir que el hilo principal termine naturalmente.

## Tiempo

```java
long milis = System.currentTimeMillis(); // milisegundos desde epoch (1970-01-01 UTC)
long nanos = System.nanoTime();          // nanosegundos desde un origen arbitrario
```

`currentTimeMillis()` es util para marcas de tiempo absolutas. Depende del reloj del sistema y puede verse afectado por ajustes de hora.

`nanoTime()` esta disenado para medir intervalos de tiempo transcurrido, no para tiempo absoluto. Usa el contador de alto rendimiento (TSC) del procesador cuando esta disponible. Es mas preciso que `currentTimeMillis()` pero no esta relacionado con el tiempo del calendario.

```java
long inicio = System.nanoTime();
tareaLarga();
long duracion = System.nanoTime() - inicio;
System.out.println("Duracion: " + duracion + " ns");
```

## Garbage collector

```java
System.gc(); // sugerir ejecucion del GC
```

`System.gc()` no fuerza la recoleccion de basura, sino que sugiere a la JVM que seria un buen momento para ejecutarla. La JVM puede ignorar la sugerencia (`-XX:+DisableExplicitGC`). Su uso excesivo puede degradar el rendimiento.

```java
System.runFinalization(); // sugerir ejecucion de finalizadores
```

## Variables de entorno

```java
Map<String, String> env = System.getenv();
String path = System.getenv("PATH");
```

`System.getenv()` devuelve un mapa no modificable de las variables de entorno del sistema operativo. Especificar el nombre como argumento devuelve el valor o `null` si no existe.

## Propiedades del sistema

```java
String version = System.getProperty("java.version");
String home = System.getProperty("java.home");
String userDir = System.getProperty("user.dir");
String lineSep = System.getProperty("line.separator");
String fileSep = System.getProperty("file.separator");
String osName = System.getProperty("os.name");
```

`System.getProperty(String key)` lee propiedades de la JVM. `System.getProperty(String key, String default)` permite un valor por defecto.

Establecer propiedades:

```java
System.setProperty("mi.aplicacion.config", "produccion");
String config = System.getProperty("mi.aplicacion.config", "desarrollo");
```

Las propiedades del sistema pueden pasarse al iniciar la JVM con `-D`:

```bash
java -Dmi.aplicacion.config=produccion -jar app.jar
```

## Copia de arrays

```java
int[] origen = {1, 2, 3, 4, 5};
int[] destino = new int[5];
System.arraycopy(origen, 0, destino, 0, origen.length);
```

`System.arraycopy(Object src, int srcPos, Object dest, int destPos, int length)` copia un rango de elementos de un array fuente a un array destino. Es una operacion nativa optimizada que puede ser significativamente mas rapida que un bucle manual. Lanza `NullPointerException` si algun array es nulo, `IndexOutOfBoundsException` si los indices son invalidos, y `ArrayStoreException` si los tipos son incompatibles.

Para copia mas flexible (con soporte para sublistas y tipos genericos), `java.util.Arrays.copyOf()` y `java.util.Arrays.copyOfRange()` envuelven `System.arraycopy()`.

## Identidad del objeto

```java
Object obj = new Object();
int hash = System.identityHashCode(obj);
```

`System.identityHashCode()` devuelve el codigo hash basado en la direccion de memoria del objeto, independientemente de si `hashCode()` ha sido sobrescrito. Es el mismo valor que usa `IdentityHashMap`.

## Seguridad

```java
SecurityManager sm = System.getSecurityManager();
if (sm != null) {
    sm.checkPermission(new java.io.FilePermission("/tmp/-", "read"));
}
```

`System.getSecurityManager()` devuelve el gestor de seguridad actual (si existe). Desde Java 17, `SecurityManager` esta deprecado para eliminacion.
