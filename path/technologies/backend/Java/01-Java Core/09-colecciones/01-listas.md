# Listas

La interfaz `List` del paquete `java.util` representa una secuencia ordenada de elementos que permite acceso posicional, inserción en cualquier indice y elementos duplicados. Es una de las estructuras de datos mas utilizadas en Java y cuenta con varias implementaciones, cada una con características de rendimiento y comportamiento distintos.

## Interfaz List

`List` extiende `Collection` y define un contrato donde cada elemento tiene una posición (indice). Los indices comienzan en 0.

```java
List<String> lista = new ArrayList<>();
```

## Metodos principales

### Insercion y eliminacion

```java
lista.add("Java");               // al final
lista.add(0, "Primero");         // en posicion especifica
lista.addAll(Arrays.asList("A", "B"));
String eliminado = lista.remove(0);    // por indice
boolean exito = lista.remove("Java");  // por objeto (primera ocurrencia)
lista.clear();                         // elimina todos
```

### Acceso y busqueda

```java
String elemento = lista.get(0);           // acceso por indice
int indice = lista.indexOf("Java");       // primera ocurrencia (-1 si no existe)
int ultimo = lista.lastIndexOf("Java");   // ultima ocurrencia
boolean contiene = lista.contains("Java"); // busqueda por objeto
boolean vacia = lista.isEmpty();
int tam = lista.size();
```

### Modificacion y sublistas

```java
lista.set(0, "Python");                  // reemplaza en posicion
List<String> sub = lista.subList(1, 4);  // vista entre indices [1,4)
Collections.sort(lista);                 // ordenar
```

> `subList()` devuelve una vista de la lista original. Las modificaciones en la sublista se reflejan en la original. Modificar la lista original (estructuralmente) después de crear una sublista invalida la sublista y lanza `ConcurrentModificationException`.

## ArrayList

`ArrayList` es la implementación mas usada. Internamente usa un array dinámico que se redimensiona automáticamente cuando se supera su capacidad.

```java
List<String> arrayList = new ArrayList<>();          // capacidad inicial 10
List<String> conCapacidad = new ArrayList<>(100);    // capacidad inicial 100
List<String> desdeOtra = new ArrayList<>(otraLista); // copia
```

### Rendimiento

| Operacion     | Coste           | Notas                             |
| ------------- | --------------- | --------------------------------- |
| `add(e)`      | O(1) amortizado | Redimensión ocasional O(n)        |
| `add(i, e)`   | O(n)            | Desplaza elementos a la derecha   |
| `get(i)`      | O(1)            | Acceso directo por indice         |
| `set(i, e)`   | O(1)            | Reemplazo directo                 |
| `remove(i)`   | O(n)            | Desplaza elementos a la izquierda |
| `remove(o)`   | O(n)            | Búsqueda lineal + desplazamiento  |
| `contains(o)` | O(n)            | Búsqueda lineal                   |
| `indexOf(o)`  | O(n)            | Búsqueda lineal                   |

### Redimensión

Cuando el array interno se llena, `ArrayList` crea un nuevo array con capacidad `(capacidadActual * 3/2) + 1` y copia todos los elementos. Este proceso es O(n), pero ocurre con poca frecuencia, dando un coste amortizado O(1) para inserción al final.

```java
// Forzar capacidad para evitar redimensiones frecuentes
((ArrayList<String>) lista).ensureCapacity(1000);
// Reducir capacidad al tamanio actual
((ArrayList<String>) lista).trimToSize();
```

## LinkedList

`LinkedList` implementa `List` y `Deque` mediante una lista doblemente enlazada. Cada nodo contiene el elemento, una referencia al nodo anterior y otra al siguiente.

```java
List<String> linkedList = new LinkedList<>();
Deque<String> comoDeque = new LinkedList<>();
```

### Rendimiento

| Operacion       | Coste | Notas                             |
| --------------- | ----- | --------------------------------- |
| `add(e)`        | O(1)  | Insercion al final                |
| `add(i, e)`     | O(n)  | Navegacion hasta posicion         |
| `get(i)`        | O(n)  | Debe recorrer desde el inicio/fin |
| `set(i, e)`     | O(n)  | Navegacion + reemplazo            |
| `remove(i)`     | O(n)  | Navegacion + eliminacion          |
| `remove(o)`     | O(n)  | Busqueda lineal                   |
| `addFirst(e)`   | O(1)  | Solo disponible como Deque        |
| `removeFirst()` | O(1)  | Solo disponible como Deque        |
| `getFirst()`    | O(1)  | Solo disponible como Deque        |

### Optimizacion de busqueda

`LinkedList` implementa una busqueda indexada que decide si comenzar desde la cabecera o desde la cola segun la proximidad del indice:

```java
// Internamente: si indice < (size/2), recorre desde inicio; si no, desde fin
E get(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++) x = x.next;
        return x.item;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) x = x.prev;
        return x.item;
    }
}
```

## Vector

`Vector` es la version sincronizada de `ArrayList`, introducida en Java 1.0. Todos sus metodos son `synchronized`, lo que introduce sobrecarga innecesaria en contextos monohilo.

```java
List<String> vector = new Vector<>();
```

En la practica, `Vector` esta obsoleto. Para concurrencia se prefiere `Collections.synchronizedList(new ArrayList<>())` o `CopyOnWriteArrayList`.

## ArrayList vs LinkedList

Usar `ArrayList` cuando:

- Se necesita acceso aleatorio frecuente (`get()`, `set()`).
- Predominan inserciones/eliminaciones al final de la lista.
- Se conoce el tamaño aproximado de antemano.
- La sobrecarga de memoria por elemento debe ser minima (solo el array).

Usar `LinkedList` cuando:

- Se necesitan inserciones/eliminaciones frecuentes al inicio o en medio.
- Se usa la lista como cola o deque (FIFO, LIFO).
- El acceso secuencial predomina sobre el aleatorio.
- La cantidad de elementos es impredecible y variable.

```java
// ArrayList: rápido en acceso, lento en inserción media
List<Integer> al = new ArrayList<>();
for (int i = 0; i < 100000; i++) al.add(i);

// LinkedList: rápido en inserción al inicio
List<Integer> ll = new LinkedList<>();
for (int i = 0; i < 100000; i++) ll.addFirst(i);
```

En la mayoría de casos prácticos, `ArrayList` es la opcion correcta por su menor overhead de memoria y mejor localidad de cache. `LinkedList` tiene sentido solo en escenarios especificos de insercion/eliminacion frecuente en posiciones no finales.


