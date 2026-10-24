# Mapas

La interfaz `Map<K,V>` del paquete `java.util` modela una estructura de datos que asocia claves unicas a valores. No extiende `Collection`; tiene su propia jerarquia de interfaces e implementaciones. Cada clave puede mapear a un solo valor, y las claves se comparan mediante `equals()` (en `HashMap`) o `Comparable`/`Comparator` (en `TreeMap`).

## Interfaz Map

```java
Map<String, Integer> mapa = new HashMap<>();
```

## Metodos principales

### Insercion y actualizacion

```java
mapa.put("clave", 42);              // inserta o reemplaza si existe
mapa.putIfAbsent("clave", 0);       // solo si no existe
mapa.putAll(otroMapa);              // copia todas las entradas
```

### Obtencion y verificacion

```java
Integer valor = mapa.get("clave");            // null si no existe
Integer valorDef = mapa.getOrDefault("clave", -1); // con valor por defecto
boolean existe = mapa.containsKey("clave");
boolean valorExiste = mapa.containsValue(42);
boolean vacio = mapa.isEmpty();
int tam = mapa.size();
```

### Eliminacion

```java
Integer eliminado = mapa.remove("clave");            // elimina y devuelve valor
boolean eliminadoCond = mapa.remove("clave", 42);    // solo si clave=valor
mapa.clear();                                        // elimina todo
```

### Vistas de coleccion

```java
Set<String> claves = mapa.keySet();           // vista de claves
Collection<Integer> valores = mapa.values();  // vista de valores
Set<Map.Entry<String, Integer>> entradas = mapa.entrySet(); // vista de pares
```

Las vistas reflejan cambios en el mapa original. Modificar la vista (excepto `clear()` y `remove()`) lanza `UnsupportedOperationException`.

```java
for (Map.Entry<String, Integer> entry : mapa.entrySet()) {
    String k = entry.getKey();
    Integer v = entry.getValue();
    entry.setValue(v * 2); // modifica el valor en el mapa original
}
```

### Metodos de computo (Java 8+)

```java
mapa.compute("clave", (k, v) -> v == null ? 1 : v + 1);
mapa.computeIfAbsent("clave", k -> k.length());
mapa.computeIfPresent("clave", (k, v) -> v * 2);
mapa.merge("clave", 1, Integer::sum); // suma si existe, 1 si no
```

## HashMap

`HashMap` es la implementacion mas usada. Internamente usa un array de buckets donde cada bucket es una estructura que puede ser una lista enlazada o un arbol rojo-negro (desde Java 8).

```java
Map<String, Integer> hashMap = new HashMap<>();
Map<String, Integer> conCapacidad = new HashMap<>(1000, 0.75f);
```

### Rendimiento

| Operacion | Coste promedio | Notas |
|-----------|----------------|-------|
| `put(k, v)` | O(1) | Con buena distribucion de hash |
| `get(k)` | O(1) | Con buena distribucion de hash |
| `remove(k)` | O(1) | Con buena distribucion de hash |
| `containsKey(k)` | O(1) | Con buena distribucion de hash |

### Colisiones y arbolizacion

Cuando multiples claves tienen el mismo hash, quedan en el mismo bucket. Hasta 8 elementos, el bucket es una lista enlazada (busqueda O(n)). A partir de 8, la lista se convierte en un arbol rojo-negro (busqueda O(log n)). Esto protege contra ataques de colision hash.

```java
// Factores internos
static final int DEFAULT_INITIAL_CAPACITY = 16;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
```

### Requisitos para las claves

- Implementar `hashCode()` correctamente (consistente, distribucion uniforme).
- Implementar `equals()` correctamente.
- Ser inmutables o no modificar los campos usados en `hashCode()` mientras estan en el mapa.

> `HashMap` permite una clave `null` (se almacena en el bucket 0). `Hashtable` no permite claves ni valores `null`.

## LinkedHashMap

`LinkedHashMap` extiende `HashMap` y mantiene una lista doblemente enlazada que preserva el orden de insercion (por defecto) o el orden de acceso.

```java
Map<String, Integer> linkedMap = new LinkedHashMap<>();
linkedMap.put("Z", 1);
linkedMap.put("A", 2);
linkedMap.put("M", 3);
// Iteracion: Z, A, M (orden de insercion)
```

### Orden de acceso

```java
Map<String, Integer> accessOrder = new LinkedHashMap<>(16, 0.75f, true);
accessOrder.put("A", 1);
accessOrder.put("B", 2);
accessOrder.get("A"); // "A" se mueve al final
// Iteracion: B, A (orden de ultimo acceso)
```

Util para implementar caches LRU:

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LRUCache(int maxSize) {
        super(16, 0.75f, true);
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}
```

## TreeMap

`TreeMap` implementa `NavigableMap` y `SortedMap`. Almacena las claves ordenadas segun su orden natural o un `Comparator`.

```java
Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("Z", 3);
treeMap.put("A", 1);
treeMap.put("M", 2);
// Iteracion: A, M, Z (orden natural)
```

### Metodos de NavigableMap

```java
TreeMap<Integer, String> tm = new TreeMap<>();
tm.put(1, "uno"); tm.put(3, "tres"); tm.put(5, "cinco");

Map.Entry<Integer, String> floor = tm.floorEntry(4);   // 3=tres
Map.Entry<Integer, String> ceil = tm.ceilingEntry(4);  // 5=cinco
Integer menorKey = tm.lowerKey(3);     // 1
Integer mayorKey = tm.higherKey(3);    // 5
Map<Integer, String> sub = tm.subMap(2, 6); // {3=tres, 5=cinco}
Map<Integer, String> head = tm.headMap(3);  // {1=uno}
Map<Integer, String> tail = tm.tailMap(3);  // {3=tres, 5=cinco}
```

### Rendimiento

| Operacion | Coste |
|-----------|-------|
| `put(k, v)` | O(log n) |
| `get(k)` | O(log n) |
| `remove(k)` | O(log n) |
| `containsKey(k)` | O(log n) |

## Hashtable

`Hashtable` es la clase historica sincronizada (Java 1.0). Todos sus metodos son `synchronized`.

```java
Map<String, Integer> hashtable = new Hashtable<>();
```

Esta obsoleta para codigo nuevo. Para concurrencia se prefiere `ConcurrentHashMap`. No permite claves ni valores `null`.

## ConcurrentHashMap

`ConcurrentHashMap` esta disenado para acceso concurrente con alto rendimiento. No bloquea toda la estructura; usa segment locking y, desde Java 8, operaciones basadas en CAS (Compare-And-Swap) sobre nodos individuales.

```java
Map<String, Integer> concurrente = new ConcurrentHashMap<>();
concurrente.put("clave", 1);
Integer valor = concurrente.get("clave");
```

```java
// Operaciones atomicas
concurrente.computeIfAbsent("clave", k -> inicializar(k));
concurrente.merge("contador", 1, Integer::sum);
```

`ConcurrentHashMap` no permite `null` como clave ni como valor (porque en un entorno concurrente, `get()` devolver `null` podria significar "no existe" o "valor nulo", y no hay forma segura de distinguir).

## Mapas inmutables

```java
Map<String, Integer> inmutable = Map.of("A", 1, "B", 2);
Map<String, Integer> copiaInmutable = Collections.unmodifiableMap(hashMap);
Map<String, Integer> copiaDefensiva = Map.copyOf(hashMap);
```

`Map.of()` y `Map.copyOf()` (Java 9+) crean mapas inmutables que no aceptan null.
