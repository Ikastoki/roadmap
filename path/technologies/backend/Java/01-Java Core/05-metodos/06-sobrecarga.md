# Sobrecarga

La sobrecarga (overloading) permite que una misma clase tenga varios metodos con el mismo nombre pero diferente lista de parametros. Es una forma de polimorfismo estatico (compile-time polymorphism) donde el compilador determina que version ejecutar basandose en los tipos y cantidad de argumentos en la llamada.

## Mismo nombre, diferentes parametros

Para que dos metodos esten sobrecargados, deben diferenciarse en al menos uno de estos aspectos:

- Numero de parametros
- Tipo de los parametros
- Orden de los parametros

El tipo de retorno no cuenta para la sobrecarga. Dos metodos con el mismo nombre y parametros pero distinto retorno no compilan.

```java
public class Calculadora {

    // Mismo nombre, diferente numero de parametros
    public int sumar(int a, int b) {
        return a + b;
    }

    public int sumar(int a, int b, int c) {
        return a + b + c;
    }

    // Mismo nombre, diferente tipo de parametros
    public double sumar(double a, double b) {
        return a + b;
    }

    // Mismo nombre, diferente orden
    public String formatear(String texto, int veces) {
        return texto.repeat(veces);
    }

    public String formatear(int veces, String texto) {
        return texto.repeat(veces);
    }
}
```

## Resolucion en tiempo de compilacion

La sobrecarga se resuelve en compile-time. El compilador examina los tipos de los argumentos en la llamada y selecciona la version mas especifica que coincida.

```java
Calculadora calc = new Calculadora();

calc.sumar(2, 3);        // llama a sumar(int, int)
calc.sumar(2, 3, 4);     // llama a sumar(int, int, int)
calc.sumar(2.5, 3.7);    // llama a sumar(double, double)
```

El compilador sigue un proceso de busqueda escalonado: primero busca una coincidencia exacta, luego widening primitivo, luego autoboxing, y finalmente varargs. Si encuentra mas de una candidata aplicable en el mismo nivel, se produce un error de ambiguedad.

```java
public void metodo(int a, double b) { }
public void metodo(double a, int b) { }

// metodo(5, 10); // ERROR: ambiguedad, ambas son igual de especificas
```

## Ejemplo con System.out.println

El caso mas ubicuo de sobrecarga en Java es `System.out.println()`, que tiene sobrecargas para cada tipo primitivo, `String`, `char[]`, `Object`, y mas.

```java
System.out.println(42);        // println(int)
System.out.println(3.14);      // println(double)
System.out.println("texto");   // println(String)
System.out.println(true);      // println(boolean)
System.out.println(new Object()); // println(Object)
```

Sin sobrecarga, cada tipo de dato requeriria un metodo con nombre distinto (`printlnInt`, `printlnDouble`, etc.), lo que haria la API considerablemente menos coherente.

## Sobrecarga de constructores

Los constructores tambien se sobrecargan, permitiendo crear objetos de distintas formas segun los datos disponibles.

```java
public class Pedido {
    private String id;
    private String cliente;
    private double total;

    public Pedido(String cliente, double total) {
        this.id = UUID.randomUUID().toString();
        this.cliente = cliente;
        this.total = total;
    }

    public Pedido(String id, String cliente, double total) {
        this.id = id;
        this.cliente = cliente;
        this.total = total;
    }
}
```

La sobrecarga de constructores se combina frecuentemente con `this()` para evitar duplicacion, tal como se vio en el tema anterior.

## Diferencia con overriding

La sobrecarga (overloading) y la sobrescritura (overriding) son conceptos que suelen confundirse. Sus diferencias fundamentales son:

| Aspecto           | Overloading                     | Overriding                       |
|-------------------|---------------------------------|----------------------------------|
| Resolucion        | Compile-time                    | Runtime                          |
| Clases            | Misma clase o jerarquia         | Subclase respecto a superclase   |
| Firma             | Diferente lista de parametros   | Misma firma exacta               |
| Retorno           | Puede variar                    | Covariante (mismo o subtipo)     |
| Modificador       | Sin restriccion                 | No menos restrictivo             |

```java
public class Base {
    public void mostrar(String s) { }      // metodo base
    public void mostrar(int i) { }         // sobrecarga en la misma clase
}

public class Derivada extends Base {
    public void mostrar(String s) { }      // OVERRIDING: misma firma
    public void mostrar(double d) { }      // SOBRECARGA en la subclase
}
```

El overriding se refiere a redefinir un metodo con la misma firma en una subclase para proporcionar comportamiento especifico. La sobrecarga simplemente anade nuevas versiones del metodo con diferentes parametros, ya sea en la misma clase o en una subclase.
