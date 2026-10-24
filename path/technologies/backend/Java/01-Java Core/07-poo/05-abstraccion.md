# Abstraccion

La abstraccion consiste en modelar entidades del mundo real u operaciones logicas mediante tipos que exponen unicamente los detalles relevantes, ocultando la complejidad interna. Java proporciona dos mecanismos principales para lograr abstraccion: las clases abstractas y las interfaces. Ambos permiten definir contratos que las clases concretas deben cumplir.

## Clases abstractas

Una clase abstracta se declara con la palabra clave `abstract` y no puede instanciarse directamente. Sirve como base para otras clases, definiendo metodos abstractos (sin implementacion) que las subclases deben implementar, y opcionalmente metodos concretos con implementacion comun.

```java
public abstract class Figura {
    protected String color;

    public Figura(String color) {
        this.color = color;
    }

    // Metodo abstracto: sin cuerpo, debe implementarse en subclases
    public abstract double calcularArea();

    // Metodo concreto: implementacion compartida
    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    // Metodo concreto que usa el metodo abstracto
    public void mostrarInfo() {
        System.out.println("Figura de color " + color
            + " con area " + calcularArea());
    }
}

public class Circulo extends Figura {
    private double radio;

    public Circulo(String color, double radio) {
        super(color);
        this.radio = radio;
    }

    @Override
    public double calcularArea() {
        return Math.PI * radio * radio;
    }
}

public class Rectangulo extends Figura {
    private double ancho;
    private double alto;

    public Rectangulo(String color, double ancho, double alto) {
        super(color);
        this.ancho = ancho;
        this.alto = alto;
    }

    @Override
    public double calcularArea() {
        return ancho * alto;
    }
}
```

Una clase abstracta puede tener constructores, campos de instancia, metodos concretos, y metodos estaticos. La unica restriccion es que no puede crearse con `new`.

> Una clase que contenga al menos un metodo abstracto debe declararse como abstracta. Una clase abstracta puede tener cero metodos abstractos y aun asi no ser instanciable, lo cual es util para clases base que solo deben usarse mediante herencia.

## Interfaces

Una interfaz en Java declara un contrato que las clases implementadoras deben cumplir. Desde Java 8, las interfaces pueden contener metodos `default` y `static`. Desde Java 9, tambien metodos `private`.

```java
public interface Imprimible {
    // Metodo abstracto (implicitamente public abstract)
    void imprimir();

    // Metodo default (Java 8+)
    default void imprimirConEncabezado(String encabezado) {
        System.out.println("=== " + encabezado + " ===");
        imprimir();
        System.out.println("=== FIN ===");
    }

    // Metodo estatico (Java 8+)
    static Imprimible crearPantalla() {
        return () -> System.out.println("Impresion en pantalla");
    }

    // Metodo privado (Java 9+)
    private void log(String mensaje) {
        System.out.println("[LOG] " + mensaje);
    }
}

public class Documento implements Imprimible {
    private String contenido;

    public Documento(String contenido) {
        this.contenido = contenido;
    }

    @Override
    public void imprimir() {
        System.out.println(contenido);
    }
}
```

Una clase puede implementar multiples interfaces, lo que proporciona una forma de polimorfismo multiple que Java no permite con herencia de clases.

```java
public interface Serializable {
    String serializar();
}

public interface Exportable {
    void exportar(String formato);
}

public class Informe implements Imprimible, Serializable, Exportable {
    private String titulo;
    private String cuerpo;

    @Override
    public void imprimir() {
        System.out.println(titulo + ": " + cuerpo);
    }

    @Override
    public String serializar() {
        return "{\"titulo\":\"" + titulo + "\",\"cuerpo\":\"" + cuerpo + "\"}";
    }

    @Override
    public void exportar(String formato) {
        System.out.println("Exportando a " + formato + "...");
    }
}
```

## Diferencias entre clase abstracta e interfaz

| Aspecto              | Clase abstracta                     | Interfaz                            |
|----------------------|-------------------------------------|-------------------------------------|
| Instanciacion        | No permitida                        | No permitida                        |
| Metodos abstractos   | `abstract`                          | Implicitos (sin `default` o `static`) |
| Metodos concretos    | Si                                  | Solo `default`, `static`, `private` |
| Campos               | Cualquier tipo y acceso             | Solo `public static final`          |
| Constructores        | Si                                  | No                                  |
| Herencia multiple    | No (extends unico)                  | Si (implements multiples)           |
| Palabra clave        | `abstract class`                    | `interface`                         |
| Estado               | Puede tener estado (campos)         | No tiene estado                     |

## Cuando usar cada una

**Clase abstracta** cuando las clases comparten estado (campos) y comportamiento parcial. Es adecuada para jerarquias donde las subclases son variantes de un concepto base: `Vehiculo` con `marca`, `modelo`, `acelerar()`.

**Interfaz** cuando se quiere definir un contrato de comportamiento que puede ser implementado por clases no relacionadas jerarquicamente. Las interfaces representan capacidades: `Comparable`, `Runnable`, `Serializable`.

## Programar a interfaces

El principio de "programar a interfaces, no a implementaciones" establece que las variables y parametros deben declararse con el tipo mas abstracto posible (generalmente una interfaz), no con la clase concreta. Esto desacopla el codigo de implementaciones especificas.

```java
// MAL: acoplado a ArrayList
public ArrayList<String> getNombres() {
    return new ArrayList<>();
}

// BIEN: programado a interfaz
public List<String> getNombres() {
    return new ArrayList<>(); // puede cambiarse a LinkedList sin afectar API
}

// MAL: parametro concreto
public void procesar(ArrayList<String> lista) { }

// BIEN: parametro abstracto
public void procesar(List<String> lista) { }
```

Este principio permite cambiar la implementacion sin modificar el codigo que la usa. Es la base de patrones como Strategy, Factory, y del polimorfismo en general. El codigo queda mas flexible, testeable y mantenible.
