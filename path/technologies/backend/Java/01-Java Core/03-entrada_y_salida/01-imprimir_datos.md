# Imprimir datos

La salida de datos es la forma en que un programa comunica resultados al usuario. En Java, la clase `System` proporciona un campo estático `out` de tipo `PrintStream` que se utiliza como canal estándar de salida. A través de este objeto se invocan los métodos `println`, `print` y `printf`, cada uno con un comportamiento distinto respecto al formato y el salto de línea.

## System.out.println

El método `println` imprime el argumento recibido seguido de un salto de línea. Acepta cualquier tipo de dato primitivo u objeto, ya que posee sobrecargas para `int`, `double`, `boolean`, `char`, `String`, entre otros. Cuando se le pasa un objeto, Internamente invoca `String.valueOf()` y luego imprime el resultado.

```java
public class EjemploPrintln {
    public static void main(String[] args) {
        System.out.println("Hola, mundo");
        System.out.println(42);
        System.out.println(3.14);
        System.out.println(true);
    }
}
```

Salida:

```
Hola, mundo
42
3.14
true
```

Cada llamada a `println` genera una línea nueva en la consola. Esto resulta útil para registrar información de depuración o mostrar resultados de forma legible, aunque en producción se preferirá un framework de logging adecuado.

## System.out.print

El método `print` es idéntico a `println` excepto en un aspecto: no añade el salto de línea al final. El cursor queda posicionado justo después del último carácter impreso. Esto permite construir salidas compuestas paso a paso.

```java
public class EjemploPrint {
    public static void main(String[] args) {
        System.out.print("Nombre: ");
        System.out.print("Carlos");
        System.out.print(" | Edad: ");
        System.out.println(28);
    }
}
```

Salida:

```
Nombre: Carlos | Edad: 28
```

La combinación de `print` y `println` da control fino sobre la presentación. Sin embargo, para salidas complejas resulta más práctico emplear `printf` o la concatenación con `+`.

## System.out.printf

El método `printf` permite imprimir con formato siguiendo la convención de `printf` de C. Utiliza *format specifiers* (especificadores de formato) que se sustituyen por los valores de los argumentos. Se trata del mecanismo más potente para controlar la apariencia de la salida numérica y de texto.

```java
public class EjemploPrintf {
    public static void main(String[] args) {
        String nombre = "Ana";
        int edad = 25;
        double salario = 3250.75;

        System.out.printf("Nombre: %s, Edad: %d, Salario: %.2f%n", nombre, edad, salario);
        System.out.printf("Porcentaje: %6.1f%%%n", 85.3);
    }
}
```

Salida:

```
Nombre: Ana, Edad: 25, Salario: 3250.75
Porcentaje:   85.3%
```

> El carácter `%n` es el separador de plataforma en `printf`. A diferencia de `\n`, que es siempre salto de línea Unix, `%n` se adapta al sistema operativo (Windows usa `\r\n`).

## Concatenacion con +

El operador `+` sobre `String` realiza concatenación. Cuando uno de los operandos es un tipo primitivo, se convierte a `String` antes de la operación. La concatenación se evalúa de izquierda a derecha.

```java
public class EjemploConcatenacion {
    public static void main(String[] args) {
        String nombre = "Marta";
        int anios = 30;
        double nota = 8.5;

        String mensaje = nombre + " tiene " + anios + " años y nota " + nota;
        System.out.println(mensaje);

        System.out.println("Resultado: " + 2 + 3);
        System.out.println("Resultado: " + (2 + 3));
    }
}
```

Salida:

```
Marta tiene 30 años y nota 8.5
Resultado: 23
Resultado: 5
```

El segundo y tercer ejemplo muestran un matiz clave: `+` asocia de izquierda a derecha. En `"Resultado: " + 2 + 3` se concatenan cadenas, por lo que el resultado es `"Resultado: 23"`. Al parentizar `(2 + 3)` se fuerza la suma aritmética antes de la concatenación.

## Variables en println

La interpolación de variables dentro de cadenas es un patrón habitual. En Java no existe interpolación directa tipo `${variable}` como en otros lenguajes, pero se puede lograr de varias formas.

```java
public class EjemploVariables {
    public static void main(String[] args) {
        String producto = "Teclado";
        double precio = 45.99;
        int cantidad = 3;
        double total = precio * cantidad;

        // Concatenacion clasica
        System.out.println("Producto: " + producto);
        System.out.println("Total: " + total);

        // Text blocks (Java 15+)
        String texto = """
                Producto: %s
                Precio: %.2f
                Cantidad: %d
                Total: %.2f
                """.formatted(producto, precio, cantidad, total);
        System.out.println(texto);
    }
}
```

El método `String.formatted()` (Java 15+) es equivalente a `String.format()` pero invocado sobre la plantilla, lo que mejora la legibilidad en cadenas multilínea.

> Para Java 21+ se puede usar el `StringTemplate` (preview), que permite escribir `STR."El total es \{total}"` directamente. Sin embargo, esta característica aún se encuentra en fase de incubación.

## Buenas practicas

Cuando se trata de salida para el usuario final, `System.out.printf` ofrece el mayor control sobre formato y alineación. Para depuración rápida, `System.out.println` con concatenación es suficiente. En código de producción, el uso directo de `System.out` debe ser sustituido por un framework de logging (como SLF4J o Log4j) que permita controlar niveles, destinos y formato sin modificar el código fuente.

```java
// Depuracion rapida
System.out.println("DEBUG - valor de x: " + x);

// Salida formateada al usuario
System.out.printf("El IVA (%.0f%%) sobre %.2f es: %.2f%n", 21.0, base, iva);
```
