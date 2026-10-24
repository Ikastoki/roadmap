# Scanner

La clase `java.util.Scanner` es el mecanismo estándar para leer entrada del usuario en programas de consola. Actúa como un analizador léxico que descompone un flujo de datos en tokens, permitiendo leer enteros, decimales, cadenas y otros tipos de forma estructurada. Internamente utiliza expresiones regulares para delimitar los tokens.

## Crear un Scanner

Para leer de la entrada estándar (`System.in`) se instancia un `Scanner` pasándole el flujo de bytes de entrada. El `Scanner` se encarga de decodificar los bytes a caracteres y luego tokenizar el resultado.

```java
import java.util.Scanner;

public class EjemploScanner {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        System.out.print("Introduce tu nombre: ");
        String nombre = sc.nextLine();

        System.out.print("Introduce tu edad: ");
        int edad = sc.nextInt();

        System.out.printf("Hola %s, tienes %d anios.%n", nombre, edad);
        sc.close();
    }
}
```

> `Scanner` no cierra el flujo `System.in` al invocar `close()`. Cerrarlo es buena práctica para liberar recursos internos del `Scanner`, pero no cierra la entrada estándar.

## Metodos de lectura principales

`Scanner` ofrece distintos métodos según el tipo de dato que se desee leer. Cada método avanza el cursor interno hasta el siguiente token del tipo indicado.

| Método | Tipo de retorno | Descripción |
|---|---|---|
| `nextLine()` | `String` | Lee la línea completa hasta el salto de línea |
| `next()` | `String` | Lee el siguiente token (hasta espacio) |
| `nextInt()` | `int` | Lee el siguiente token como entero |
| `nextDouble()` | `double` | Lee el siguiente token como doble |
| `nextBoolean()` | `boolean` | Lee el siguiente token como booleano |
| `nextLong()` | `long` | Lee el siguiente token como long |

```java
import java.util.Scanner;

public class EjemploMetodos {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        System.out.print("Introduce una palabra: ");
        String palabra = sc.next();

        System.out.print("Introduce un numero entero: ");
        int entero = sc.nextInt();

        System.out.print("Introduce un decimal: ");
        double decimal = sc.nextDouble();

        System.out.printf("Palabra: %s, Entero: %d, Decimal: %.2f%n",
                palabra, entero, decimal);
        sc.close();
    }
}
```

## El problema de nextLine despues de nextInt

Un error muy frecuente es mezclar `nextLine()` con `nextInt()` o `nextDouble()`. Los métodos numéricos leen el token pero dejan el carácter de salto de línea (`\n`) en el buffer. La subsiguiente `nextLine()` lee ese `\n` vacío y devuelve una cadena vacía.

```java
import java.util.Scanner;

public class EjemploBuffer {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        System.out.print("Edad: ");
        int edad = sc.nextInt();

        // Esto no funciona como se espera:
        // sc.nextLine(); // Consumir el salto de linea pendiente
        System.out.print("Direccion: ");
        String direccion = sc.nextLine(); // Captura el \n pendiente

        System.out.printf("Edad: %d, Direccion: '%s'%n", edad, direccion);
        sc.close();
    }
}
```

Para solucionarlo, se inserta una llamada adicional a `nextLine()` justo después de leer el número, con el fin de consumir el salto de línea pendiente.

```java
System.out.print("Edad: ");
int edad = sc.nextInt();
sc.nextLine(); // Consumir el \n pendiente

System.out.print("Direccion: ");
String direccion = sc.nextLine();
```

## Validacion de entrada

`Scanner` ofrece el método `hasNext()` y sus variantes para verificar si el siguiente token coincide con el tipo esperado antes de leerlo. Esto evita `InputMismatchException`.

```java
import java.util.Scanner;

public class EjemploValidacion {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        System.out.print("Introduce un entero positivo: ");
        while (!sc.hasNextInt()) {
            System.out.print("No es un entero. Intenta de nuevo: ");
            sc.next(); // Descartar entrada invalida
        }
        int valor = sc.nextInt();

        System.out.printf("Valor valido: %d%n", valor);
        sc.close();
    }
}
```

El patrón de validación consiste en un bucle `while` que consulta `hasNextXxx()` y, si devuelve `false`, consume y descarta la entrada inválida con `next()` antes de solicitar de nuevo.

> Para validar rangos específicos (por ejemplo, edad entre 0 y 150), se puede usar `hasNextInt()` combinado con una condición adicional: `sc.hasNextInt() && sc.nextInt() >= 0 && sc.nextInt() <= 150`. Sin embargo, consumir y reintentar es más seguro.

## Patron try-with-resources

Desde Java 7, `Scanner` implementa `AutoCloseable`, por lo que puede utilizarse en un bloque `try-with-resources` para garantizar el cierre automático del recurso.

```java
import java.util.Scanner;

public class EjemploTryWithResources {
    public static void main(String[] args) {
        try (Scanner sc = new Scanner(System.in)) {
            System.out.print("Nombre: ");
            String nombre = sc.nextLine();

            System.out.print("Edad: ");
            int edad = sc.nextInt();

            System.out.printf("Hola %s, tienes %d anios.%n", nombre, edad);
        }
    }
}
```

El recurso se cierra automáticamente al finalizar el bloque `try`, tanto si el código se ejecuta correctamente como si se lanza una excepción. Esto elimina la necesidad de invocar `sc.close()` manualmente.

## Scanner con archivos

`Scanner` también puede leer desde un `File`, un `InputStream` o incluso una `String`. Esto lo convierte en una herramienta versátil para procesar distintas fuentes de datos con la misma interfaz.

```java
import java.util.Scanner;
import java.io.File;
import java.io.FileNotFoundException;

public class EjemploArchivo {
    public static void main(String[] args) {
        try (Scanner sc = new Scanner(new File("datos.txt"))) {
            while (sc.hasNextLine()) {
                String linea = sc.nextLine();
                System.out.println(linea);
            }
        } catch (FileNotFoundException e) {
            System.out.println("Archivo no encontrado: " + e.getMessage());
        }
    }
}
```

> Al leer archivos, `Scanner` puede lanzar `FileNotFoundException`, que es una excepción checked. El bloque `try-with-resources` gestiona el cierre, pero la excepción debe ser capturada o declarada con `throws`.
