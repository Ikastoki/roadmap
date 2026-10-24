# NIO: Buffers

Los buffers son el recipiente de datos en NIO. Mientras que la E/S clasica trabaja directamente con arreglos de bytes o caracteres, NIO exige que todos los datos transferidos hacia o desde un canal pasen por un buffer. Esta intermediacion no es un rodeo innecesario: proporciona un control preciso sobre cuantos datos se leen, cuantos quedan por leer, y en que posicion se encuentra la operacion actual. El buffer actua como un bloque de memoria contiguo con metadatos que describen el estado de la transferencia.

## ByteBuffer: el buffer fundamental

`ByteBuffer` es la clase central del sistema de buffers en NIO. Es un buffer de bytes que puede usarse tanto para lectura como para escritura, y constituye la contrapartida natural de los canales. Se crea mediante metodos factory estaticos, no mediante constructores.

`ByteBuffer.allocate(int capacity)` crea un buffer con memoria gestionada por la JVM en el heap. `ByteBuffer.allocateDirect(int capacity)` crea un buffer con memoria directa fuera del heap, lo que evita una copia intermedia durante las operaciones de E/S con el sistema operativo. `ByteBuffer.wrap(byte[] array)` envuelve un arreglo existente, de modo que los cambios en el buffer se reflejan en el arreglo y viceversa.

```java
ByteBuffer heapBuf    = ByteBuffer.allocate(1024);
ByteBuffer directBuf  = ByteBuffer.allocateDirect(1024);
byte[] datos         = {1, 2, 3, 4, 5};
ByteBuffer wrapBuf   = ByteBuffer.wrap(datos);
```

## Capacidad, posicion, limite y marca

Un buffer tiene cuatro propiedades que definen su estado:

**Capacity.** El numero maximo de elementos que puede contener el buffer. Se establece en la creacion y no cambia durante la vida del buffer.

**Position.** El indice del siguiente elemento a leer o escribir. Comienza en 0 tras la creacion y avanza automaticamente con cada operacion `put()` o `get()`.

**Limit.** El indice del primer elemento que no debe leerse o escribirse. En modo escritura (recien creado o tras `clear()`) coincide con la capacidad. En modo lectura (tras `flip()`) marca el final de los datos escritos.

**Mark.** Una posicion opcional que puede establecerse con `mark()` y a la que se retorna con `reset()`.

La relacion entre estas propiedades siempre debe cumplir: `0 <= mark <= position <= limit <= capacity`.

## Operaciones principales

`put(byte b)` escribe un byte en la posicion actual y avanza la posicion. `put(byte[] src)` escribe un arreglo completo. `get()` lee un byte en la posicion actual y avanza la posicion. `get(byte[] dst)` lee bytes hacia un arreglo destino.

`flip()` es la operacion de transicion del modo escritura al modo lectura: establece `limit = position` y luego `position = 0`. Invierte el sentido del buffer sin perder los datos escritos.

`clear()` prepara el buffer para escritura: establece `limit = capacity` y `position = 0`. Los datos anteriores permanecen en memoria pero seran sobrescritos.

`compact()` copia los datos no leidos al principio del buffer, establece `position` justo despues de esos datos y `limit = capacity`. Es util cuando se leyo parcialmente un buffer y se necesita seguir escribiendo sin perder lo que aun no se proceso.

`rewind()` establece `position = 0` sin modificar `limit`, permitiendo releer los datos. `mark()` guarda la posicion actual y `reset()` restaura esa posicion.

```java
ByteBuffer buf = ByteBuffer.allocate(8);
buf.put((byte) 'H');
buf.put((byte) 'e');
buf.put((byte) 'l');
buf.put((byte) 'l');
buf.put((byte) 'o');

buf.flip();
while (buf.hasRemaining()) {
    System.out.print((char) buf.get());
}
// Imprime: Hello

buf.rewind();
buf.get(); // 'H'
buf.mark();
buf.get(); // 'e'
buf.get(); // 'l'
buf.reset();
buf.get(); // 'e' (vuelve a la marca)
```

## Buffer directo vs buffer heap

La diferencia fundamental entre `allocate()` y `allocateDirect()` radica en como se gestiona la memoria y como interactua con el sistema operativo. Un buffer heap reside en el monton de la JVM y su arreglo subyacente es un `byte[]` normal. Cuando un canal realiza una operacion de E/S con un buffer heap, la JVM debe copiar temporalmente los datos a un buffer nativo fuera del heap para que el sistema operativo pueda acceder a ellos.

Un buffer directo asigna memoria nativa fuera del heap mediante `malloc()` o equivalente. El sistema operativo puede acceder directamente a esa memoria sin copias intermedias, lo que mejora el rendimiento en E/S intensiva. Sin embargo, la asignacion y liberacion de buffers directos es mas costosa que la de buffers heap, por lo que conviene reutilizarlos.

> Los buffers directos son recomendables para operaciones de E/S de larga duracion sobre canales, especialmente cuando se transfieren grandes volumenes de datos. Para buffers pequenos y operaciones esporadicas, los buffers heap son suficientes.

## Otros tipos de buffer

NIO proporciona buffers tipados para cada tipo primitivo: `CharBuffer`, `ShortBuffer`, `IntBuffer`, `LongBuffer`, `FloatBuffer` y `DoubleBuffer`. Todos heredan de `Buffer` y comparten las mismas propiedades de capacidad, posicion, limite y marca. `CharBuffer` es particularmente util para trabajar con cadenas de texto, y puede crearse mediante `CharBuffer.wrap(CharSequence)` o `CharBuffer.allocate(int)`.

```java
CharBuffer cb = CharBuffer.allocate(32);
cb.put("NIO buffers");
cb.flip();
while (cb.hasRemaining()) {
    System.out.print(cb.get());
}
```

`MappedByteBuffer` es una variante especial de `ByteBuffer` que permite mapear directamente una region de un archivo en memoria mediante `FileChannel.map()`, lo que hace que el archivo sea accesible como si estuviera completamente en RAM, delegando al sistema operativo la gestion de la paginacion.
