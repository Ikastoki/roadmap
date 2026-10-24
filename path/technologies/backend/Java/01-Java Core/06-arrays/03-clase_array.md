# Clase Array

La clase `java.util.Arrays` proporciona metodos estaticos para manipular arrays de forma comoda y eficiente. Incluye operaciones de ordenacion, busqueda, copia, relleno y comparacion, entre otras. Su uso evita tener que implementar manualmente algoritmos clasicos y garantiza un rendimiento optimo mediante implementaciones nativas y algoritmos probados.

## sort()

Ordena los elementos del array en orden ascendente. Utiliza Dual-Pivot Quicksort para tipos primitivos y TimSort para objetos, ambos con complejidad O(n log n).

```java
int[] numeros = {5, 2, 8, 1, 9};
Arrays.sort(numeros);
System.out.println(Arrays.toString(numeros)); // [1, 2, 5, 8, 9]

// Ordenacion parcial: solo ordena un rango [fromIndex, toIndex)
int[] parcial = {5, 2, 8, 1, 9};
Arrays.sort(parcial, 1, 4); // ordena indices 1, 2, 3
System.out.println(Arrays.toString(parcial)); // [5, 1, 2, 8, 9]
```

Para ordenar objetos con un criterio personalizado, se pasa un `Comparator`:

```java
String[] nombres = {"Juan", "Ana", "Pedro", "Luis"};
Arrays.sort(nombres); // orden alfabetico
Arrays.sort(nombres, (a, b) -> b.compareTo(a)); // orden descendente
```

## binarySearch()

Busca un valor en un array ordenado usando busqueda binaria. Devuelve el indice si se encuentra, o un valor negativo que indica el punto de insercion si no esta presente.

```java
int[] numeros = {1, 3, 5, 7, 9, 11};
int indice = Arrays.binarySearch(numeros, 7);  // 3
int noEncontrado = Arrays.binarySearch(numeros, 6); // -4 (insercion en indice 3)
```

> El array debe estar ordenado antes de llamar a `binarySearch()`. De lo contrario, el resultado es indeterminado. El valor de retorno negativo se transforma en punto de insercion con la formula `(-(indice) - 1)`.

## fill()

Rellena todo el array o un rango con el mismo valor.

```java
int[] buffer = new int[100];
Arrays.fill(buffer, 0); // todos los elementos a 0

int[] parcial = new int[10];
Arrays.fill(parcial, 2, 7, 42); // indices 2 a 6 con valor 42
```

`fill()` no crea un nuevo array; modifica el array existente. Es util para inicializar o resetear arrays, especialmente en procesamiento de buffers y matrices.

## copyOf() y copyOfRange()

`copyOf()` crea una copia del array con la longitud especificada. Si la nueva longitud es mayor, los elementos adicionales se rellenan con el valor por defecto del tipo. Si es menor, se truncan los elementos sobrantes.

```java
int[] original = {1, 2, 3, 4, 5};

int[] copiaCompleta = Arrays.copyOf(original, original.length); // [1, 2, 3, 4, 5]
int[] masGrande = Arrays.copyOf(original, 7);   // [1, 2, 3, 4, 5, 0, 0]
int[] masPequeno = Arrays.copyOf(original, 3);  // [1, 2, 3]
```

`copyOfRange()` copia un rango especifico del array.

```java
int[] original = {1, 2, 3, 4, 5};
int[] rango = Arrays.copyOfRange(original, 1, 4); // [2, 3, 4]
```

> A diferencia de `clone()`, `copyOf()` y `copyOfRange()` son metodos genericos que preservan el tipo del array. Son la forma recomendada de copiar arrays en Java moderno.

## toString()

Devuelve una representacion en cadena del array, mostrando sus elementos entre corchetes y separados por coma. Sin `Arrays.toString()`, imprimir un array muestra la referencia al objeto (algo como `[I@1a2b3c`).

```java
int[] numeros = {10, 20, 30};
System.out.println(Arrays.toString(numeros)); // [10, 20, 30]

String[] nombres = {"Ana", "Luis", "Sofia"};
System.out.println(Arrays.toString(nombres)); // [Ana, Luis, Sofia]

// Para arrays multidimensionales
int[][] matriz = {{1, 2}, {3, 4}};
System.out.println(Arrays.deepToString(matriz)); // [[1, 2], [3, 4]]
```

## equals()

Compara dos arrays elemento por elemento. Devuelve `true` si ambos arrays tienen la misma longitud y todos los elementos son iguales segun `equals()` (para objetos) o `==` (para primitivos).

```java
int[] a = {1, 2, 3};
int[] b = {1, 2, 3};
int[] c = {1, 2, 4};

System.out.println(Arrays.equals(a, b)); // true
System.out.println(Arrays.equals(a, c)); // false

// Para arrays multidimensionales
int[][] m1 = {{1, 2}, {3, 4}};
int[][] m2 = {{1, 2}, {3, 4}};
System.out.println(Arrays.deepEquals(m1, m2)); // true
```

> `a.equals(b)` sin `Arrays.equals()` compara referencias, no contenido. Dos arrays distintos con los mismos elementos nunca seran `equals` entre si segun el metodo de `Object`.

## asList()

Convierte un array en una `List` de tamaño fijo respaldada por el array original. Los cambios en la lista o en el array se reflejan mutuamente.

```java
String[] colores = {"rojo", "verde", "azul"};
List<String> lista = Arrays.asList(colores);

System.out.println(lista); // [rojo, verde, azul]
lista.set(0, "naranja");
System.out.println(colores[0]); // "naranja" (se refleja en el array)

// lista.add("negro"); // UnsupportedOperationException: tamaño fijo
```

`Arrays.asList()` no admite operaciones que modifiquen el tamaño (`add`, `remove`). Para una lista completamente mutable debe pasarse al constructor de `ArrayList`:

```java
List<String> mutable = new ArrayList<>(Arrays.asList(colores));
mutable.add("negro"); // correcto
```

> El metodo `asList()` es de uso tan frecuente que desde Java 9 se introdujo `List.of()` como alternativa inmutable: `List<String> lista = List.of("rojo", "verde", "azul");`. Esta no permite modificar ningun elemento.
