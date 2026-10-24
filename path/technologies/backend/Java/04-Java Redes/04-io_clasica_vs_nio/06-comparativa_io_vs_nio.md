# Comparativa IO vs NIO

La eleccion entre `java.io` y `java.nio` no es una cuestion de superioridad tecnica absoluta, sino de adecuacion al problema. Cada modelo resuelve un conjunto diferente de restricciones y ofrece compensaciones (trade-offs) distintas. Comprender estas diferencias es esencial para disenar sistemas de red que sean correctos, eficientes y mantenibles.

## IO clasico: simple, bloqueante, no escalable

El modelo clasico de `java.io` se caracteriza por su simplicidad conceptual. Un hilo lee de un stream o escribe en el, y mientras la operacion no se completa, el hilo permanece bloqueado. No hay estado que gestionar, no hay buffers que vaciar parcialmente, no hay selectores que configurar.

**Ventajas:**
- Curva de aprendizaje baja. Cualquier desarrollador con conocimientos basicos de Java puede escribir codigo de E/S funcional.
- Codigo lineal y facil de depurar. Las operaciones ocurren en el orden en que se escriben.
- Ideal para aplicaciones con pocas conexiones simultaneas (decenas, no miles).
- Rendimiento predecible para operaciones de archivo locales.

**Desventajas:**
- Cada conexion requiere un hilo dedicado. Con 10.000 conexiones, se necesitan 10.000 hilos.
- El cambio de contexto entre hilos (context switching) degrada el rendimiento cuando hay muchos hilos activos.
- La memoria de pila por hilo (tipicamente 1 MB) limita el numero maximo de conexiones.
- No hay mecanismo nativo para timeout en operaciones bloqueantes sobre ciertos streams.

```java
// Modelo clasico: un hilo por conexion
ExecutorService pool = Executors.newCachedThreadPool();
while (true) {
    Socket socket = serverSocket.accept();
    pool.submit(() -> {
        InputStream in = socket.getInputStream();
        OutputStream out = socket.getOutputStream();
        // Procesar en este hilo...
    });
}
```

## NIO: complejo, no bloqueante, escalable

NIO introduce un modelo asincrono y no bloqueante que permite a un unico hilo gestionar miles de conexiones. La contrapartida es una mayor complejidad en el codigo y una gestion de estado mas cuidadosa.

**Ventajas:**
- Escalabilidad a decenas de miles de conexiones con un numero reducido de hilos.
- Uso eficiente de la memoria del sistema, al no necesitar una pila por conexion.
- Control fino sobre las operaciones de E/S (buffers directos, posicionamiento, transferencia entre canales).
- Operaciones de archivo avanzadas (mapeo en memoria, bloqueos de archivo, transferencia directa entre canales).

**Desventajas:**
- Mayor complejidad de codigo. El bucle de eventos, los buffers parciales y la gestion de estados requieren cuidado.
- El codigo es menos legible y mas dificil de depurar que el modelo clasico.
- Para operaciones de archivo simples, NIO no ofrece ventajas de rendimiento significativas.
- El uso incorrecto de buffers (no llamar a `flip()`, olvidar `clear()`) introduce bugs sutiles.

```java
// Modelo NIO: un hilo, multiples canales con selector
Selector selector = Selector.open();
channel.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select();
    for (SelectionKey key : selector.selectedKeys()) {
        // Determinar que canal esta listo y procesarlo
    }
}
```

## Cuando usar cada uno

**Usar java.io cuando:**
- La aplicacion maneja pocas conexiones simultaneas (menos de 100).
- El codigo debe ser simple y mantenido por desarrolladores de diversa experiencia.
- Las operaciones son principalmente de archivo local, no de red.
- El tiempo de vida de cada conexion es corto y el numero de hilos es manejable.

**Usar java.nio cuando:**
- La aplicacion debe manejar miles de conexiones simultaneas (servidores de chat, APIs REST de alta concurrencia, juegos multijugador).
- Se requiere un control fino sobre el rendimiento de E/S.
- Se necesita transferir grandes volumenes de datos entre canales sin copias intermedias.
- La memoria del sistema es un recurso limitado y no se puede asignar un hilo por conexion.

## Virtual Threads como alternativa moderna

Java 21 introdujo los virtual threads (hilos virtuales) como parte de Project Loom. Un virtual thread es un hilo gestionado por la JVM que no esta vinculado a un hilo del sistema operativo. Cuando un virtual thread ejecuta una operacion bloqueante de E/S, la JVM lo suspende y reutiliza el hilo del sistema operativo para ejecutar otro virtual thread.

Esto cambia el panorama de la E/S clasica: ya no es necesario usar NIO para escalar, porque los virtual threads permiten escribir codigo bloqueante simple que escala igual que NIO. Con virtual threads, se puede retornar al modelo de un hilo por conexion con codigo lineal, pero sin el costo de los hilos del sistema operativo.

```java
// Con virtual threads, el modelo clasico vuelve a ser escalable
try (var server = new ServerSocket(8080)) {
    while (true) {
        Socket socket = server.accept();
        Thread.startVirtualThread(() -> {
            // E/S bloqueante en un virtual thread
            // La JVM gestiona el multiplexado internamente
        });
    }
}
```

> Virtual threads no hacen que NIO sea irrelevante. En escenarios donde se requiere el maximo rendimiento posible o control fino sobre buffers directos (como en servidores proxy, balanceadores de carga o frameworks de red), NIO sigue siendo la opcion preferida. Pero para la mayoria de las aplicaciones empresariales, virtual threads simplifican drasticamente el codigo sin sacrificar escalabilidad.

## Resumen comparativo

| Aspecto         | java.io                          | java.nio                              |
|-----------------|----------------------------------|---------------------------------------|
| Modelo          | Bloqueante, stream unidireccional| No bloqueante, canal bidireccional    |
| Concurrencia    | Un hilo por conexion             | Un hilo para muchas conexiones        |
| Escalabilidad   | Limitada (C10K)                  | Alta (C100K+)                         |
| Complejidad     | Baja                             | Alta                                  |
| Buffers         | Arreglos de bytes                | ByteBuffer con posicion/limite        |
| Multiplexado    | No                               | Si, mediante Selector                 |
| Archivos        | Streams basicos                  | FileChannel, MappedByteBuffer         |
| Virtual threads | Compatible y simplificado        | Sigue siendo util para maximo rendimiento |
