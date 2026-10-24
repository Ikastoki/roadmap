# Genericos

Los genericos, introducidos en Java 5, permiten parametrizar tipos. Una clase, interfaz o metodo puede operar sobre un tipo especificado por el usuario, proporcionando seguridad de tipos en tiempo de compilacion sin necesidad de casting explicito. Son fundamentales en el framework de colecciones y en la mayoria de APIs modernas de Java.

## Clases genericas

```java
public class Caja<T> {
    private T contenido;

    public void guardar(T contenido) {
        this.contenido = contenido;
    }

    public T obtener() {
        return contenido;
    }
}
```

```java
Caja<String> cajaString = new Caja<>();
cajaString.guardar("Hola");
String valor = cajaString.obtener(); // sin casting

Caja<Integer> cajaInt = new Caja<>();
cajaInt.guardar(42);
```

### Multiples parametros de tipo

```java
public class Par<K, V> {
    private K clave;
    private V valor;

    public Par(K clave, V valor) {
        this.clave = clave;
        this.valor = valor;
    }

    public K getClave() { return clave; }
    public V getValor() { return valor; }
}
```

```java
Par<String, Integer> par = new Par<>("edad", 30);
```

## Metodos genericos

Los metodos genericos tienen sus propios parametros de tipo, declarados antes del tipo de retorno.

```java
public class Utilidades {
    public static <T> T obtenerPrimero(List<T> lista) {
        return lista.get(0);
    }

    public static <T> void intercambiar(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
}
```

```java
String primero = Utilidades.<String>obtenerPrimero(lista);
// Inferencia de tipos: no es necesario especificar el tipo
String primero2 = Utilidades.obtenerPrimero(lista);
```

## Interfaces genericas

```java
public interface Procesador<T> {
    void procesar(T item);
    T resultado();
}
```

```java
public class ProcesadorString implements Procesador<String> {
    private String resultado;

    @Override
    public void procesar(String item) {
        resultado = item.toUpperCase();
    }

    @Override
    public String resultado() { return resultado; }
}
```

## Bounded types (tipos acotados)

Restringen los tipos permitidos a aquellos que extienden una clase o implementan una interfaz.

```java
// T debe ser Number o subclase
public class Calculadora<T extends Number> {
    private T valor;

    public double duplicar() {
        return valor.doubleValue() * 2;
    }
}
```

```java
Calculadora<Integer> calcInt = new Calculadora<>();
Calculadora<Double> calcDouble = new Calculadora<>();
// Calculadora<String> error: String no extiende Number
```

### Multiples bounds

```java
public class Procesador<T extends Comparable<T> & Serializable> {
    // T debe implementar Comparable y Serializable
}
```

## Wildcards (? extends T, ? super T)

Los comodines (`?`) representan tipos desconocidos. Son esenciales para flexibilidad en APIs genericas.

### Upper-bounded wildcard (? extends T)

Acepta `T` o cualquier subtipo de `T`. Sirve para lectura (covarianza).

```java
public static double sumar(Collection<? extends Number> numeros) {
    double total = 0;
    for (Number n : numeros) {
        total += n.doubleValue();
    }
    return total;
}
```

```java
List<Integer> enteros = Arrays.asList(1, 2, 3);
List<Double> dobles = Arrays.asList(1.5, 2.5, 3.5);
sumar(enteros); // OK
sumar(dobles);  // OK
```

### Lower-bounded wildcard (? super T)

Acepta `T` o cualquier supertipo de `T`. Sirve para escritura (contravarianza).

```java
public static void addNumeros(List<? super Integer> lista) {
    lista.add(1);
    lista.add(2);
}
```

```java
List<Number> nums = new ArrayList<>();
List<Object> objs = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
addNumeros(nums); // OK
addNumeros(objs); // OK
// addNumeros(ints) no compilaria si el parametro fuera List<Number>
```

### Wildcard ilimitado (?)

```java
public static void imprimir(Collection<?> coleccion) {
    for (Object o : coleccion) {
        System.out.println(o);
    }
}
```

## PECS (Producer Extends, Consumer Super)

Es la regla nemotecnica para usar wildcards correctamente:

- **Producer Extends**: si el parametro produce elementos (se leen), usa `? extends T`.
- **Consumer Super**: si el parametro consume elementos (se escriben), usa `? super T`.

```java
// Producer: solo leemos de la coleccion
public static void copiar(Collection<? extends T> fuente, ...) { }

// Consumer: solo escribimos en la coleccion
public static void llenar(Collection<? super T> destino, T elemento) { }
```

Ejemplo en `Collections.copy()`:

```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (int i = 0; i < src.size(); i++) {
        dest.set(i, src.get(i));
    }
}
```

## Type erasure

Los genericos en Java son un fenomeno de tiempo de compilacion. El compilador elimina la informacion de tipo generico y la reemplaza con casts y metodos puente (bridge methods). Este proceso se llama type erasure.

```java
// En tiempo de compilacion:
List<String> lista = new ArrayList<>();
String s = lista.get(0);

// Despues de erasure:
List lista = new ArrayList();
String s = (String) lista.get(0);
```

### Consecuencias de type erasure

- No se puede usar `instanceof` con tipos parametrizados:

```java
if (lista instanceof ArrayList<String>) {} // ERROR
if (lista instanceof ArrayList<?>) {}      // OK (wildcard)
```

- No se pueden crear arrays de tipos parametrizados:

```java
List<String>[] array = new List<String>[10]; // ERROR
List<?>[] array = new List<?>[10];           // OK
```

- No se pueden lanzar o capturar genericos:

```java
class MiExcepcion<T> extends Exception {} // ERROR
```

### Bridge methods

El compilador genera metodos puente para preservar el polimorfismo:

```java
// Original
public class MiLista implements Comparable<MiLista> {
    public int compareTo(MiLista o) { ... }
}

// Despues de erasure y bridge:
public class MiLista implements Comparable {
    public int compareTo(MiLista o) { ... }
    public int compareTo(Object o) { // bridge method
        return compareTo((MiLista) o);
    }
}
```

## Restricciones adicionales

- No se pueden crear instancias de parametros de tipo:

```java
T obj = new T(); // ERROR
```

- No se pueden crear arrays de parametros de tipo:

```java
T[] array = new T[10]; // ERROR
```

- No se pueden usar tipos primitivos como parametros de tipo:

```java
List<int> lista = new ArrayList<>(); // ERROR, usar List<Integer>
```

- Los parametros de tipo no pueden usarse en contextos estaticos:

```java
public class Caja<T> {
    private static T campo; // ERROR
    public static T metodo() { ... } // ERROR
}
```
