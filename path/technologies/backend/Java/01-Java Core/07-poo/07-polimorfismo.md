# Polimorfismo

El polimorfismo es la capacidad de un objeto de comportarse de multiples formas. En Java se manifiesta en dos variantes: el polimorfismo estatico (sobrecarga, resuelto en compilacion) y el polimorfismo dinamico (sobrescritura, resuelto en ejecucion). Ambos son esenciales para escribir codigo flexible y extensible.

## Sobrecarga (polimorfismo estatico, compile-time)

La sobrecarga permite que varios metodos compartan el mismo nombre pero con diferentes parametros. El compilador selecciona la version adecuada en tiempo de compilacion basandose en los tipos de los argumentos.

```java
public class Calculadora {
    public int sumar(int a, int b) {
        return a + b;
    }

    public double sumar(double a, double b) {
        return a + b;
    }

    public int sumar(int a, int b, int c) {
        return a + b + c;
    }
}

Calculadora calc = new Calculadora();
calc.sumar(2, 3);       // compile-time: sumar(int, int)
calc.sumar(2.5, 3.7);   // compile-time: sumar(double, double)
calc.sumar(1, 2, 3);    // compile-time: sumar(int, int, int)
```

La sobrecarga se resuelve antes de que el programa se ejecute. No tiene impacto en el rendimiento en runtime. El compilador tambien realiza widening y autoboxing para encontrar la coincidencia mas cercana.

## Sobrescritura (polimorfismo dinamico, runtime)

La sobrescritura ocurre cuando una subclase redefine un metodo de la superclase con la misma firma. La JVM selecciona la implementacion a ejecutar en tiempo de ejecucion segun la clase real del objeto, no segun el tipo de la referencia.

```java
public class Animal {
    public void sonido() {
        System.out.println("Sonido generico");
    }
}

public class Perro extends Animal {
    @Override
    public void sonido() {
        System.out.println("Guau");
    }
}

public class Gato extends Animal {
    @Override
    public void sonido() {
        System.out.println("Miau");
    }
}

public class Main {
    public static void main(String[] args) {
        Animal a1 = new Perro();
        Animal a2 = new Gato();

        a1.sonido(); // "Guau" (runtime)
        a2.sonido(); // "Miau" (runtime)
    }
}
```

Aunque las variables `a1` y `a2` son de tipo `Animal`, la JVM invoca el metodo de la clase real del objeto. Esto es posible gracias a la tabla de metodos virtuales (vtable) que cada clase mantiene.

## Binding estatico vs dinamico

El binding es el proceso de asociar una llamada a metodo con su implementacion.

**Binding estatico (early binding):** ocurre en compilacion. Se aplica a metodos `static`, metodos `final`, metodos `private`, y metodos sobrecargados. El compilador conoce exactamente que metodo se ejecutara.

```java
class Ejemplo {
    private void privado() { }        // binding estatico
    public final void finalMethod() { } // binding estatico
    public static void estatico() { }   // binding estatico
}
```

**Binding dinamico (late binding):** ocurre en runtime. Se aplica a metodos de instancia sobrescritos. La JVM determina la implementacion concreta durante la ejecucion.

```java
Animal a = new Perro();
a.sonido(); // binding dinamico: se evalua en runtime
```

## Casting de objetos

El casting entre tipos de una jerarquia permite tratar un objeto como un tipo mas especifico o mas general.

**Upcasting:** convertir un objeto a un tipo mas general en la jerarquia. Siempre es seguro y se hace implicitamente.

```java
Perro perro = new Perro();
Animal animal = perro; // upcasting implicito
Object obj = perro;    // upcasting a Object
```

**Downcasting:** convertir un objeto a un tipo mas especifico. Requiere casting explicito y puede lanzar `ClassCastException` si el objeto no es realmente de ese tipo.

```java
Animal animal = new Perro();
Perro perro = (Perro) animal;  // downcasting valido
perro.ladrar();

Animal a = new Gato();
// Perro p = (Perro) a; // ClassCastException en runtime
```

```java
if (animal instanceof Perro) {
    Perro p = (Perro) animal; // seguro gracias a instanceof
    p.ladrar();
}
```

Desde Java 16, el pattern matching de `instanceof` simplifica este patron:

```java
if (animal instanceof Perro p) {
    p.ladrar(); // p ya esta casteado dentro del bloque
}
```

## Upcasting y downcasting

El upcasting es siempre seguro porque restringe la interfaz visible: solo se pueden llamar metodos de la superclase. El downcasting es peligroso porque amplia la interfaz y el objeto podria no tener esa capacidad.

```java
public class Main {
    public static void procesar(Animal animal) {
        animal.sonido(); // solo metodos de Animal

        if (animal instanceof Perro p) {
            p.ladrar(); // downcasting solo si es necesario
        }
    }

    public static void main(String[] args) {
        procesar(new Perro()); // upcasting implicito al pasar argumento
        procesar(new Gato());
    }
}
```

> El downcasting sin `instanceof` es una mala practica. Siempre debe verificarse el tipo real antes de hacer downcasting para evitar `ClassCastException`. La necesidad frecuente de downcasting puede indicar que la abstraccion no es la adecuada.

## Polimorfismo con interfaces

Las interfaces son el vehiculo mas potente de polimorfismo en Java. Una misma interfaz puede tener implementaciones completamente distintas.

```java
public interface Reproducible {
    void reproducir();
}

public class Musica implements Reproducible {
    @Override
    public void reproducir() {
        System.out.println("Reproduciendo musica...");
    }
}

public class Video implements Reproducible {
    @Override
    public void reproducir() {
        System.out.println("Reproduciendo video...");
    }
}

public class Reproductor {
    public void ejecutar(Reproducible r) {
        r.reproducir(); // polimorfismo: cualquier Reproducible
    }
}
```

Este nivel de polimorfismo permite que el codigo que usa la interfaz ignore completamente la implementacion concreta. Nuevos tipos pueden anadirse sin modificar el codigo existente, cumpliendo el principio abierto/cerrado (Open/Closed Principle).
