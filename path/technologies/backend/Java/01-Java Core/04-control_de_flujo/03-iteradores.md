# Iteradores

Un iterador proporciona una interfaz uniforme para recorrer elementos de una colección sin exponer su representación interna. En Java, la interfaz `java.util.Iterator` define el contrato para recorrer cualquier colección que implemente `Iterable`, estableciendo un patrón que desacopla el código que consume datos del código que los almacena.

## La interfaz Iterator

`Iterator<E>` declara tres métodos: `hasNext()` para verificar si quedan elementos, `next()` para obtener el siguiente, y `remove()` para eliminar el último elemento devuelto.

```java
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class EjemploIterator {
    public static void main(String[] args) {
        List<String> nombres = new ArrayList<>();
        nombres.add("Ana");
        nombres.add("Carlos");
        nombres.add("Maria");
        nombres.add("Pedro");

        Iterator<String> it = nombres.iterator();
        while (it.hasNext()) {
            String nombre = it.next();
            System.out.println(nombre);
        }
    }
}
```

`hasNext()` no avanza el cursor; solo consulta. `next()` avanza el cursor y devuelve el elemento. Si se llama a `next()` sin que `hasNext()` devuelva `true`, se lanza `NoSuchElementException`.

## Eliminar elementos con Iterator

El método `remove()` elimina el último elemento devuelto por `next()`. Intentar eliminar directamente con `list.remove()` durante un bucle for-each genera `ConcurrentModificationException`, ya que la colección detecta una modificación no autorizada por el iterador.

```java
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class EjemploRemove {
    public static void main(String[] args) {
        List<Integer> numeros = new ArrayList<>();
        numeros.add(1);
        numeros.add(2);
        numeros.add(3);
        numeros.add(4);
        numeros.add(5);
        numeros.add(6);

        Iterator<Integer> it = numeros.iterator();
        while (it.hasNext()) {
            int n = it.next();
            if (n % 2 == 0) {
                it.remove(); // Elimina el par
            }
        }

        System.out.println("Impares: " + numeros);
    }
}
```

Salida:

```
Impares: [1, 3, 5]
```

> El iterador mantiene un puntero interno. Tras llamar a `next()`, el cursor queda entre el elemento devuelto y el siguiente. `remove()` elimina ese elemento sin necesidad de una segunda llamada a `next()`.

## Iterable y el bucle for-each

Cualquier clase que implemente la interfaz `Iterable<E>` puede utilizarse en un bucle for-each. La interfaz declara un único método: `iterator()`, que devuelve un `Iterator<E>`.

```java
import java.util.Iterator;

public class ColeccionPersonalizada implements Iterable<String> {
    private String[] datos = {"Primero", "Segundo", "Tercero"};

    @Override
    public Iterator<String> iterator() {
        return new Iterator<String>() {
            private int indice = 0;

            @Override
            public boolean hasNext() {
                return indice < datos.length;
            }

            @Override
            public String next() {
                return datos[indice++];
            }
        };
    }

    public static void main(String[] args) {
        ColeccionPersonalizada coleccion = new ColeccionPersonalizada();

        // for-each funciona gracias a Iterable
        for (String valor : coleccion) {
            System.out.println(valor);
        }
    }
}
```

Java desazucara el bucle for-each en una llamada a `iterator()` y un bucle `while` con `hasNext()`/`next()`. Por tanto, implementar `Iterable` es el único requisito para que una colección personalizada funcione con la sintaxis for-each.

## ListIterator para iteracion bidireccional

`ListIterator<E>` extiende `Iterator` y añade capacidad de recorrido en ambas direcciones: `hasPrevious()`, `previous()`, `nextIndex()`, `previousIndex()`, y `set(E)` para reemplazar el último elemento visitado.

```java
import java.util.ArrayList;
import java.util.List;
import java.util.ListIterator;

public class EjemploListIterator {
    public static void main(String[] args) {
        List<String> ciudades = new ArrayList<>();
        ciudades.add("Madrid");
        ciudades.add("Barcelona");
        ciudades.add("Valencia");

        // Recorrido hacia adelante
        ListIterator<String> it = ciudades.listIterator();
        System.out.println("Adelante:");
        while (it.hasNext()) {
            int idx = it.nextIndex();
            String ciudad = it.next();
            System.out.printf("  [%d] %s%n", idx, ciudad);
        }

        // Recorrido hacia atras
        System.out.println("Atras:");
        while (it.hasPrevious()) {
            int idx = it.previousIndex();
            String ciudad = it.previous();
            System.out.printf("  [%d] %s%n", idx, ciudad);
        }

        // Reemplazar el primer elemento
        it.next(); // Avanza a "Madrid"
        it.set("Sevilla");
        System.out.println("Modificado: " + ciudades);
    }
}
```

Salida:

```
Adelante:
  [0] Madrid
  [1] Barcelona
  [2] Valencia
Atras:
  [2] Valencia
  [1] Barcelona
  [0] Madrid
Modificado: [Sevilla, Barcelona, Valencia]
```

`ListIterator` se obtiene con `list.listIterator()` (cursor al inicio) o `list.listIterator(indice)` (cursor en la posición especificada). El método `set()` reemplaza el elemento que se acaba de devolver con `next()` o `previous()`.

> `ListIterator` es la herramienta adecuada para modificar colecciones durante la iteración sin `ConcurrentModificationException`. Sin embargo, para operaciones simples de filtrado, `removeIf()` (Java 8+) suele ser más conciso.
