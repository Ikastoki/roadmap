# Final

La palabra reservada `final` en Java tiene multiples aplicaciones que comparten un mismo proposito: establece que algo no puede ser modificado una vez definido. Su semantica concreta depende del contexto en que se aplique: variables, metodos, clases, o combinaciones como campos de arrays.

## final en variables

Una variable declarada como `final` solo puede asignarse una vez. Cualquier intento de reasignacion posterior produce un error de compilacion.

```java
final int MAXIMO = 100;
// MAXIMO = 200; // ERROR: no se puede reasignar

final String NOMBRE;
NOMBRE = "Sistema"; // primera y unica asignacion valida
// NOMBRE = "Otro"; // ERROR
```

En variables locales, `final` puede ser util para documentar que el valor no cambiara. Un parametro de metodo puede ser `final`, lo que impide reasignarlo dentro del metodo.

```java
public void procesar(final int id, final String nombre) {
    // id = 5; // ERROR: parametro final
}
```

Una variable local o parametro que se usa dentro de una clase anonima o lambda debe ser effectively final o estar declarada como `final`.

## final en metodos

Un metodo `final` no puede ser sobrescrito (overridden) por las subclases. Esto se usa para fijar un comportamiento que no debe alterarse en la jerarquia de herencia.

```java
public class Procesador {
    public final void iniciar() {
        System.out.println("Iniciando proceso...");
        ejecutarLogica();
        System.out.println("Proceso finalizado.");
    }

    protected void ejecutarLogica() {
        // metodo que las subclases pueden sobrescribir
    }
}

public class ProcesadorEspecifico extends Procesador {
    // @Override
    // public void iniciar() { } // ERROR: metodo final

    @Override
    protected void ejecutarLogica() {
        System.out.println("Logica especifica");
    }
}
```

Los metodos `final` son especialmente relevantes en frameworks y librerias donde ciertos metodos plantilla (template methods) deben preservar su estructura invariable. La JVM tambien puede inlinearlos mas agresivamente como optimizacion.

## final en clases

Una clase `final` no puede ser extendida. Nadie puede heredar de ella. Esta es la restriccion mas fuerte que ofrece `final`.

```java
public final class Constantes {
    public static final double PI = 3.14159;
}

// public class MisConstantes extends Constantes { } // ERROR: clase final
```

Las clases `final` son comunes en la JDK: `String`, `Integer`, `Double`, `Math`, `System`, `URL` son todas final. La inmutabilidad de `String` depende de que la clase sea final, evitando que subclases maliciosas alteren su comportamiento.

```java
// Clases inmutables requieren ser final
public final class Punto {
    private final int x;
    private final int y;

    public Punto(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }
    public int getY() { return y; }
}
```

## Arrays finales

Declarar un array como `final` impide reasignar la referencia, pero no hace inmutable el contenido del array. Los elementos individuales pueden modificarse libremente.

```java
final int[] valores = {1, 2, 3};

valores[0] = 100;     // permitido: modifica el contenido
valores[1] = 200;     // permitido

System.out.println(Arrays.toString(valores)); // [100, 200, 3]

// valores = new int[]{4, 5, 6}; // ERROR: no se puede reasignar la referencia
```

Esta distincion es frecuente fuente de confusion. `final` sobre un array protege la variable, no los datos. Para lograr un array verdaderamente inmutable hay que combinar `final` con la ausencia de metodos que modifiquen el array y no exponer la referencia interna.

```java
public class DatosSeguros {
    private final int[] datos;

    public DatosSeguros(int[] datos) {
        this.datos = Arrays.copyOf(datos, datos.length); // copia defensiva
    }

    public int[] getDatos() {
        return Arrays.copyOf(datos, datos.length); // copia defensiva al devolver
    }

    public int getElemento(int indice) {
        return datos[indice];
    }
}
```

> La combinacion `private final` en un campo array evita que la referencia se escape o se reasigne, pero el contenido sigue siendo mutable desde dentro de la clase. Por eso las copias defensivas (defensive copies) son necesarias para mantener la inmutabilidad real.

## static final

La combinacion `static final` crea constantes de clase verdaderas. Al ser `static`, el campo existe a nivel de clase, no de instancia. Al ser `final`, su valor no cambia.

```java
public class Config {
    public static final int MAX_USUARIOS = 1000;
    public static final String VERSION = "2.1.0";
    public static final LocalDate FECHA_LANZAMIENTO = LocalDate.of(2024, 6, 1);
}
```

Cuando se trata de tipos primitivos o `String`, el compilador trata estas constantes como compile-time constants y reemplaza su uso por el valor literal en el bytecode. Para objetos, la referencia es constante pero el estado del objeto puede ser mutable si la clase lo permite.
