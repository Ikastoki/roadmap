# Conjuntos

La interfaz `Set` del paquete `java.util` modela la nocion matematica de conjunto: una coleccion que no contiene elementos duplicados. Cada elemento puede aparecer como maximo una vez, y la determinacion de igualdad se basa en los metodos `equals()` y `hashCode()` del elemento.

## Interfaz Set

`Set` extiende `Collection` sin aniadir nuevos metodos; su semantica se define por la ausencia de duplicados. No garantiza orden a menos que la implementacion concreta lo especifique.

```java
Set<String> set = new HashSet<>();
```

## HashSet

`HashSet` es la implementacion mas rapida. Internamente usa un `HashMap` como respaldo: los elementos se almacenan como claves del mapa, y un objeto `Object` constante como valor. Depende de `hashCode()` y `equals()`.

```java
Set<String> hashSet = new HashSet<>();
hashSet.add("Java");
hashSet.add("Python");
hashSet.add("Java"); // ignorado, no hay duplicados
```

### Rendimiento

| Operacion | Coste |
|-----------|-------|
| `add(e)` | O(1) promedio |
| `remove(e)` | O(1) promedio |
| `contains(e)` | O(1) promedio |
| `size()` | O(1) |

### Factores de rendimiento

- **Capacidad inicial**: numero de buckets (por defecto 16).
- **Factor de carga**: 0.75 por defecto. Cuando el tamano supera `capacidad * factorDeCarga`, se redimensiona (todos los elementos se rehashean).

```java
Set<String> set = new HashSet<>(1000, 0.8f); // capacidad inicial 1000, factor 0.8
```

### Contrato hashCode() y equals()

Para que `HashSet` funcione correctamente:

1. Si `a.equals(b)` entonces `a.hashCode() == b.hashCode()`.
2. Si `a.hashCode() != b.hashCode()` entonces `!a.equals(b)`.
3. `hashCode()` debe ser consistente durante la vida del objeto (mismos campos inmutables).

```java
public class Persona {
    private final String dni;
    private final String nombre;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Persona)) return false;
        Persona p = (Persona) o;
        return dni.equals(p.dni);
    }

    @Override
    public int hashCode() {
        return Objects.hash(dni);
    }
}
```

> Si se anade un objeto a un `HashSet` y luego se modifican los campos usados en `hashCode()`, el objeto no podra ser eliminado correctamente porque su hash habra cambiado.

## LinkedHashSet

`LinkedHashSet` extiende `HashSet` y mantiene una lista doblemente enlazada que preserva el orden de insercion. El coste adicional es minimo (ligero aumento de memoria).

```java
Set<String> linkedHashSet = new LinkedHashSet<>();
linkedHashSet.add("Z");
linkedHashSet.add("A");
linkedHashSet.add("M");
// Iteracion: Z, A, M (orden de insercion)
```

```java
for (String s : linkedHashSet) {
    System.out.println(s); // Z, A, M
}
```

Tiene el mismo rendimiento que `HashSet` para `add()`, `remove()`, `contains()`, excepto que la iteracion es ligeramente mas rapida (recorre la lista enlazada en lugar de todos los buckets).

## TreeSet

`TreeSet` implementa `NavigableSet` y `SortedSet`. Mantiene los elementos ordenados segun su orden natural (`Comparable`) o un `Comparator` proporcionado. Internamente usa un `TreeMap` (arbol rojo-negro).

```java
Set<String> treeSet = new TreeSet<>();
treeSet.add("Z");
treeSet.add("A");
treeSet.add("M");
// Iteracion: A, M, Z (orden natural alfabetico)
```

```java
// Con Comparator personalizado
Set<Integer> descendente = new TreeSet<>(Comparator.reverseOrder());
descendente.addAll(Arrays.asList(3, 1, 5, 2, 4));
// Iteracion: 5, 4, 3, 2, 1
```

### Metodos de NavigableSet

```java
TreeSet<Integer> nums = new TreeSet<>(Arrays.asList(1, 3, 5, 7, 9));
int menor = nums.floor(6);     // 5 (mayor <= 6)
int mayor = nums.ceiling(6);   // 7 (menor >= 6)
int menorEstricto = nums.lower(5);   // 3 (< 5)
int mayorEstricto = nums.higher(5);  // 7 (> 5)
Integer primero = nums.pollFirst();  // 1 (extrae y elimina el primero)
Integer ultimo = nums.pollLast();    // 9 (extrae y elimina el ultimo)
Set<Integer> sub = nums.subSet(3, 8); // [3, 5, 7] (rango)
```

### Rendimiento

| Operacion | Coste |
|-----------|-------|
| `add(e)` | O(log n) |
| `remove(e)` | O(log n) |
| `contains(e)` | O(log n) |
| `first()`/`last()` | O(log n) |

## Cuando usar cada implementacion

| Implementacion | Orden | Rendimiento | Nulls | Uso tipico |
|----------------|-------|-------------|-------|------------|
| `HashSet` | No garantizado | O(1) promedio | Permite 1 null | Operaciones rapidas sin requisito de orden |
| `LinkedHashSet` | Insercion | O(1) promedio | Permite 1 null | Preservar orden de insercion |
| `TreeSet` | Natural/Comparator | O(log n) | No permite null (Comparable) | Elementos ordenados o rangos |

```java
// Sin orden especifico, maxima velocidad
Set<String> rapido = new HashSet<>();

// Preservar orden de insercion
Set<String> ordenInsercion = new LinkedHashSet<>();

// Elementos siempre ordenados
Set<String> ordenado = new TreeSet<>();

// Elementos ordenados con criterio propio
Set<Persona> porEdad = new TreeSet<>(Comparator.comparingInt(Persona::edad));
```

> `TreeSet` requiere que los elementos implementen `Comparable` o que se proporcione un `Comparator`. Si no se cumple, lanza `ClassCastException` en tiempo de ejecucion.

## Sets inmutables

```java
Set<String> inmutable = Set.of("A", "B", "C");
Set<String> copiaInmutable = Collections.unmodifiableSet(hashSet);
```

`Set.of()` (Java 9+) crea conjuntos inmutables que no aceptan elementos nulos.
