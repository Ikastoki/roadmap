# Colecciones sincronizadas

El paquete `java.util` proporciona varias estrategias para trabajar con colecciones en entornos concurrentes. La eleccion entre coleccion sincronizada y concurrente depende del patron de acceso y los requisitos de rendimiento. Las colecciones del marco `java.util.concurrent` estan disenadas desde cero para alto rendimiento en concurrencia, mientras que los envoltorios sincronizados de `Collections` anaden sincronizacion sobre colecciones existentes.

## Collections.synchronizedList/Set/Map

La clase utilitaria `Collections` ofrece metodos que envuelven una coleccion existente con un decorador que sincroniza todos sus metodos.

```java
List<String> lista = Collections.synchronizedList(new ArrayList<>());
Set<String> conjunto = Collections.synchronizedSet(new HashSet<>());
Map<String, Integer> mapa = Collections.synchronizedMap(new HashMap<>());
```

### Mecanismo interno

Cada metodo de la coleccion envuelta adquiere un monitor (`synchronized`) sobre el objeto wrapper. Es equivalente a:

```java
public boolean add(E e) {
    synchronized (mutex) { return c.add(e); }
}
```

### Iteracion manual

La sincronizacion de los metodos individuales no protege la iteracion compuesta. El iterador debe sincronizarse explicitamente:

```java
List<String> lista = Collections.synchronizedList(new ArrayList<>());

synchronized (lista) {
    Iterator<String> it = lista.iterator();
    while (it.hasNext()) {
        System.out.println(it.next());
    }
}
```

Sin el bloque `synchronized`, la iteracion puede lanzar `ConcurrentModificationException` si otro hilo modifica la lista.

```java
// INCORRECTO: puede lanzar ConcurrentModificationException
for (String s : lista) {
    if (s.equals("X")) lista.remove(s);
}
```

```java
// CORRECTO: iteracion sincronizada
synchronized (lista) {
    Iterator<String> it = lista.iterator();
    while (it.hasNext()) {
        String s = it.next();
        if (s.equals("X")) it.remove();
    }
}
```

### Limitaciones

- Un unico bloqueo para toda la estructura: bajo rendimiento en alta concurrencia.
- Las iteraciones, aunque sean solo lectura, bloquean toda la coleccion.
- Las operaciones compuestas (get-and-put, check-then-act) requieren sincronizacion externa adicional.

```java
// Operacion compuesta que requiere sincronizacion externa
synchronized (mapa) {
    if (!mapa.containsKey("clave")) {
        mapa.put("clave", valorInicializado());
    }
}
```

## CopyOnWriteArrayList

`CopyOnWriteArrayList` es una variante thread-safe de `ArrayList` donde todas las operaciones de modificacion crean una copia fresca del array subyacente. Las operaciones de lectura operan sobre la copia actual sin bloqueo.

```java
List<String> cowList = new CopyOnWriteArrayList<>();
cowList.add("A");
cowList.add("B");
String elemento = cowList.get(0); // sin bloqueo
```

### Mecanismo

En cada `add()`, `set()`, `remove()`:

1. Se bloquea con `ReentrantLock`.
2. Se copia el array completo (`Arrays.copyOf()`).
3. Se modifica la copia.
4. Se reemplaza la referencia volatil del array.

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### Caracteristicas

- Lectura sin bloqueo: `get()`, `iterator()`, `contains()` no necesitan sincronizacion.
- Iteradores seguros: el iterador usa una "instantanea" del array en el momento de su creacion, por lo que nunca lanza `ConcurrentModificationException`.
- Coste alto en escritura: cada modificacion copia todo el array, O(n).
- Uso ideal: pocas escrituras, muchas lecturas, iteraciones frecuentes.

```java
// Ideal para listas de listeners, configuracion, etc.
List<EventListener> listeners = new CopyOnWriteArrayList<>();
// Las lecturas en iteracion son seguras sin sincronizacion externa
for (EventListener l : listeners) {
    l.onEvent(event);
}
```

## ConcurrentHashMap

`ConcurrentHashMap` es la alternativa thread-safe a `HashMap` mas utilizada. Desde Java 8, usa una arquitectura de nodos y operaciones CAS mas eficiente que el segmentation locking de Java 7.

```java
Map<String, Integer> concurrente = new ConcurrentHashMap<>();
concurrente.put("clave", 1);
int valor = concurrente.get("clave");
```

### Ventajas

- Lecturas completamente sin bloqueo en la mayoria de casos.
- Escrituras con bloqueo fino solo sobre el bucket afectado.
- Operaciones atomicas compuestas integradas:

```java
concurrente.computeIfAbsent("clave", k -> inicializar(k));
concurrente.compute("clave", (k, v) -> v == null ? 1 : v + 1);
concurrente.merge("contador", 1, Integer::sum);
```

### Rendimiento

| Operacion | synchronizedMap | ConcurrentHashMap |
|-----------|----------------|-------------------|
| `get(k)` | Bloquea todo | Sin bloqueo |
| `put(k, v)` | Bloquea todo | Bloqueo por bucket |
| `size()` | Bloquea todo | O(1) aproximado |
| Iteracion | Bloquea todo | Instantanea concurrente |

## CopyOnWriteArraySet

Es un `Set` respaldado por `CopyOnWriteArrayList`. Utiliza el mismo mecanismo de copia en escritura.

```java
Set<String> cowSet = new CopyOnWriteArraySet<>();
cowSet.add("A");
```

## ConcurrentLinkedQueue y ConcurrentLinkedDeque

Colas y deques basados en algoritmos no bloqueantes (Michael-Scott, CAS). No usan bloqueos.

```java
Queue<String> cola = new ConcurrentLinkedQueue<>();
cola.offer("tarea");
String t = cola.poll();
```

## Cuando usar colecciones sincronizadas

```java
// Escenario 1: Baja concurrencia, pocos accesos
List<String> sincronizada = Collections.synchronizedList(new ArrayList<>());

// Escenario 2: Muchas lecturas, pocas escrituras, iteracion frecuente
List<String> copyOnWrite = new CopyOnWriteArrayList<>();

// Escenario 3: Alta concurrencia, mapa compartido
Map<String, Integer> concurrente = new ConcurrentHashMap<>();

// Escenario 4: Cola productor-consumidor
Queue<String> colaConcurrente = new ConcurrentLinkedQueue<>();
```

| Situacion | Recomendacion |
|-----------|---------------|
| Baja concurrencia, colecciones pequenas | `Collections.synchronized*` |
| Muchas lecturas, pocas escrituras, iteraciones | `CopyOnWriteArrayList` / `CopyOnWriteArraySet` |
| Alta concurrencia, mapas grandes | `ConcurrentHashMap` |
| Colas productor-consumidor | `ConcurrentLinkedQueue` / `LinkedBlockingQueue` |
| Operaciones atomicas compuestas | `ConcurrentHashMap.compute*` |

> Las colecciones sincronizadas con `Collections.synchronized*` son adecuadas para entornos con poca competencia. En sistemas con alta concurrencia, las colecciones del paquete `java.util.concurrent` ofrecen escalabilidad muy superior.
