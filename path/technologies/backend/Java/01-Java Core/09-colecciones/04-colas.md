# Colas

Las colas en Java se modelan mediante la interfaz `Queue` y su extension `Deque`, ambas en `java.util`. Representan colecciones disenadas para contener elementos antes de ser procesados, tipicamente en orden FIFO (first-in-first-out) o LIFO (last-in-first-out, como pila).

## Queue

`Queue<E>` extiende `Collection` y anade metodos para insercion, extraccion e inspeccion. Existen dos familias de metodos: una que lanza excepcion ante fallos y otra que devuelve valores especiales (`null` o `false`).

| Operacion | Lanza excepcion | Valor especial |
|-----------|----------------|----------------|
| Insercion | `add(e)` | `offer(e)` |
| Extraccion | `remove()` | `poll()` |
| Inspeccion | `element()` | `peek()` |

```java
Queue<String> cola = new LinkedList<>();

// Insercion
cola.offer("primero");
cola.offer("segundo");
cola.offer("tercero");

// Inspeccion sin eliminar
String frente = cola.peek(); // "primero" (null si vacia)
String frenteEx = cola.element(); // "primero" (NoSuchElementException si vacia)

// Extraccion
String primero = cola.poll(); // "primero" (null si vacia)
String primeroEx = cola.remove(); // "segundo" (NoSuchElementException si vacia)
```

## PriorityQueue

`PriorityQueue` es una cola donde los elementos se ordenan segun su orden natural o un `Comparator`, no por orden de llegada. Internamente usa un heap binario (monticulo).

```java
Queue<Integer> pq = new PriorityQueue<>();
pq.offer(30);
pq.offer(10);
pq.offer(20);
System.out.println(pq.poll()); // 10 (el menor)
System.out.println(pq.poll()); // 20
System.out.println(pq.poll()); // 30
```

```java
// Cola de maxima prioridad (orden inverso)
Queue<Integer> maxPq = new PriorityQueue<>(Comparator.reverseOrder());
maxPq.offer(30);
maxPq.offer(10);
maxPq.offer(20);
System.out.println(maxPq.poll()); // 30
```

### Rendimiento

| Operacion | Coste |
|-----------|-------|
| `offer(e)` | O(log n) |
| `poll()` | O(log n) |
| `peek()` | O(1) |
| `remove(o)` | O(n) |

La iteracion de `PriorityQueue` no garantiza orden; solo `poll()` y `peek()` acceden al elemento con mayor prioridad.

```java
PriorityQueue<Tarea> tareas = new PriorityQueue<>(Comparator.comparingInt(Tarea::prioridad));
tareas.offer(new Tarea("critica", 1));
tareas.offer(new Tarea("normal", 5));
tareas.offer(new Tarea("urgente", 2));

while (!tareas.isEmpty()) {
    Tarea t = tareas.poll(); // procesa por prioridad ascendente
    System.out.println(t.nombre());
}
```

## Deque y ArrayDeque

`Deque<E>` (double-ended queue) permite insercion y extraccion en ambos extremos. `ArrayDeque` es la implementacion mas eficiente, mejor que `LinkedList` para uso como cola o pila.

```java
ArrayDeque<String> deque = new ArrayDeque<>();
```

### Metodos de Deque

| Operacion | Lanza excepcion | Valor especial |
|-----------|----------------|----------------|
| Insercion inicio | `addFirst(e)` | `offerFirst(e)` |
| Insercion fin | `addLast(e)` | `offerLast(e)` |
| Extraccion inicio | `removeFirst()` | `pollFirst()` |
| Extraccion fin | `removeLast()` | `pollLast()` |
| Inspeccion inicio | `getFirst()` | `peekFirst()` |
| Inspeccion fin | `getLast()` | `peekLast()` |

```java
Deque<String> deque = new ArrayDeque<>();
deque.addFirst("inicio");
deque.addLast("fin");
deque.offerFirst("antes del inicio");
String primero = deque.pollFirst(); // "antes del inicio"
String ultimo = deque.pollLast();   // "fin"
```

### Uso como cola (FIFO)

```java
Deque<String> colaFifo = new ArrayDeque<>();
colaFifo.offer("primero");
colaFifo.offer("segundo");
String elem = colaFifo.poll(); // "primero"
```

### Uso como pila (LIFO)

```java
Deque<String> pila = new ArrayDeque<>();
pila.push("primero");
pila.push("segundo"); // apilar
String cima = pila.pop(); // "segundo" (desapilar)
String verCima = pila.peek(); // "primero"
```

`Deque` es preferible a la clase legacy `Stack` porque:

- `ArrayDeque` no esta sincronizado, evitando sobrecarga innecesaria.
- No extiende `Vector`, por lo que no hereda metodos que no pertenecen a una pila.
- Es mas rapido en todas las operaciones.

## Rendimiento de ArrayDeque

| Operacion | Coste |
|-----------|-------|
| `addFirst(e)` / `offerFirst(e)` | O(1) amortizado |
| `addLast(e)` / `offerLast(e)` | O(1) amortizado |
| `removeFirst()` / `pollFirst()` | O(1) |
| `removeLast()` / `pollLast()` | O(1) |
| `getFirst()` / `peekFirst()` | O(1) |
| `getLast()` / `peekLast()` | O(1) |

`ArrayDeque` usa un array circular que se redimensiona cuando se llena. Para la mayoria de casos, es la implementacion de `Deque` mas equilibrada, superando a `LinkedList` en velocidad y consumo de memoria.

```java
// Capacidad inicial
ArrayDeque<Integer> dequeCapacidad = new ArrayDeque<>(1000);
```

## Eleccion de implementacion

| Implementacion | Estructura | Caracteristicas |
|---------------|------------|-----------------|
| `LinkedList` | Lista doblemente enlazada | Implementa `List` y `Deque`; mayor overhead de memoria |
| `ArrayDeque` | Array circular | Mas rapida, menor overhead; no permite `null` |
| `PriorityQueue` | Heap binario | Ordena por prioridad; no es FIFO |
| `ConcurrentLinkedDeque` | Lista enlazada no bloqueante | Thread-safe sin bloqueos |

```java
// Cola FIFO estandar
Queue<String> fifo = new ArrayDeque<>();

// Pila LIFO
Deque<String> lifo = new ArrayDeque<>();

// Cola prioritaria
Queue<Tarea> prioritaria = new PriorityQueue<>();

// Cola concurrente
Queue<String> concurrente = new ConcurrentLinkedQueue<>();
```
