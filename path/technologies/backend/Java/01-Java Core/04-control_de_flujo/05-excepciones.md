# Excepciones

Java utiliza un mecanismo de manejo de errores basado en excepciones: objetos que encapsulan información sobre una situación anómala y que se propagan por la pila de llamadas hasta ser capturados o直至 llegar al nivel superior del programa. Este modelo sustituye los códigos de error numéricos y permite separar la lógica normal del manejo de fallos de forma estructurada.

## Jerarquia de excepciones

Todas las excepciones en Java derivan de `java.lang.Throwable`. Bajo ella se ramifican dos ramas principales: `Error` y `Exception`. `Error` representa fallos graves del entorno de ejecución que la aplicación no debe intentar recuperar (como `OutOfMemoryError` o `StackOverflowError`). `Exception` agrupa las situaciones异常 que un programa puede y debe manejar.

Dentro de `Exception`, se distinguen dos categorías fundamentales: las *checked exceptions* (excepciones verificadas) y las *unchecked exceptions* (excepciones no verificadas). Las unchecked son todas las que heredan de `RuntimeException` y sus subclases.

```
Throwable
├── Error (no se captura)
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── RuntimeException (unchecked)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ArithmeticException
    │   └── IllegalArgumentException
    ├── IOException (checked)
    ├── SQLException (checked)
    └── ...
```

## Checked vs unchecked

Las *checked exceptions* son aquellas que el compilador obliga a capturar o declarar. Si un método invoca una operación que puede lanzar una checked exception, el método debe incluir `throws` en su firma o capturar la excepción con `try-catch`. Esto fuerza al programador a considerar explícitamente los fallos posibles.

Las *unchecked exceptions* (herederas de `RuntimeException`) no requieren declaración ni captura. Son indicativas de errores de programación que deberían corregirse, como acceder a un índice fuera de rango o invocar un método sobre `null`.

```java
public class EjemploChecked {
    // Declara la excepcion porque FileNotFoundException es checked
    public static void leerArchivo(String ruta) throws java.io.FileNotFoundException {
        java.io.File archivo = new java.io.File(ruta);
        if (!archivo.exists()) {
            throw new java.io.FileNotFoundException("No existe: " + ruta);
        }
        System.out.println("Archivo encontrado");
    }

    public static void main(String[] args) {
        // Debe capturar o declarar
        try {
            leerArchivo("datos.txt");
        } catch (java.io.FileNotFoundException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

> Las checked exceptions fuerzan a tratar los errores de E/S, red y bases de datos de forma explícita. Algunos frameworks modernos (como Spring) tienden a usar unchecked exceptions para evitar el *propagating throws* en cadenas de llamadas.

## try-catch-finally

El bloque `try` envuelve el código que puede lanzar excepciones. Cada `catch` captura un tipo específico de excepción. El bloque `finally` se ejecuta siempre, tanto si se lanza una excepción como si no.

```java
public class EjemploTryCatch {
    public static void main(String[] args) {
        try {
            int[] nums = {1, 2, 3};
            System.out.println(nums[5]);
            int resultado = 10 / 0;
            System.out.println(resultado);
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Indice fuera de rango: " + e.getMessage());
        } catch (ArithmeticException e) {
            System.out.println("Error aritmetico: " + e.getMessage());
        } finally {
            System.out.println("Bloque finally ejecutado");
        }
    }
}
```

El orden de los `catch` importa. Se deben capturar las excepciones más específicas primero. Si se coloca un `catch` para una superclase antes que para una subclase, el compilador rechazará el código porque el segundo `catch` sería inalcanzable.

```java
try {
    // codigo
} catch (RuntimeException e) {
    // Esto captura ArithmeticException, NullPointerException, etc.
} catch (ArithmeticException e) {
    // NUNCA se ejecutaria, ya que ArithmeticException ya fue capturada arriba
}
```

## Multi-catch

Java 7 introdujo el multi-catch, que permite capturar múltiples tipos de excepción en un solo bloque `catch`, separándolos con `|`.

```java
public class EjemploMultiCatch {
    public static void procesar(String[] args, int indice) {
        try {
            int valor = Integer.parseInt(args[indice]);
            System.out.println("Valor: " + valor);
        } catch (NumberFormatException | ArrayIndexOutOfBoundsException e) {
            System.out.println("Error de entrada: " + e.getMessage());
        }
    }

    public static void main(String[] args) {
        procesar(new String[]{"10", "20"}, 1);
        procesar(new String[]{"abc"}, 0);
        procesar(new String[]{"10"}, 5);
    }
}
```

> En un multi-catch, la variable capturada es implícitamente `final`. No se puede reasignar dentro del bloque `catch`.

## try-with-resources

Java 7 introdujo `try-with-resources`, que cierra automáticamente cualquier objeto que implemente `AutoCloseable` al finalizar el bloque. Esto reemplaza el patrón tedioso de `try-finally` con cierre manual.

```java
import java.io.*;

public class EjemploAutoClose {
    public static void main(String[] args) {
        try (BufferedReader br = new BufferedReader(new FileReader("datos.txt"));
             PrintWriter pw = new PrintWriter(new FileWriter("salida.txt"))) {
            String linea;
            while ((linea = br.readLine()) != null) {
                pw.println(linea.toUpperCase());
            }
        } catch (IOException e) {
            System.out.println("Error de E/S: " + e.getMessage());
        }
    }
}
```

Desde Java 9, si se declara un recurso con `var`, se pueden reutilizar variables existentes:

```java
var recurso = new BufferedReader(new FileReader("datos.txt"));
try (recurso) {
    // Usar recurso
}
```

> Si se lanzan múltiples excepciones al cerrar los recursos, las de cierre se añaden como *suppressed exceptions* y se acceden con `getSuppressed()` en la excepción principal.

## Throw y throws

`throw` es una sentencia que lanza una excepción explícitamente. `throws` es una cláusula en la firma del método que declara las excepciones que el método puede propagar al invocador.

```java
public class EjemploThrowThrows {
    static void validarEdad(int edad) throws IllegalArgumentException {
        if (edad < 0 || edad > 150) {
            throw new IllegalArgumentException("Edad fuera de rango: " + edad);
        }
        System.out.println("Edad valida: " + edad);
    }

    public static void main(String[] args) {
        try {
            validarEdad(25);
            validarEdad(-5);
        } catch (IllegalArgumentException e) {
            System.out.println("Validacion fallida: " + e.getMessage());
        }
    }
}
```

Una excepción propagada con `throws` debe ser capturada en algún punto de la cadena de llamadas o declarada en `main` (lo que termina el programa con el *stack trace*).

## Excepciones personalizadas

Crear excepciones propias permite modelar errores específicos del dominio de la aplicación. Se recomienda heredar de `RuntimeException` para excepciones no verificadas o de `Exception` para verificadas.

```java
public class SaldoInsuficienteException extends RuntimeException {
    private final double saldoActual;
    private final double cantidadSolicitada;

    public SaldoInsuficienteException(double saldoActual, double cantidadSolicitada) {
        super(String.format("Saldo insuficiente: disponible %.2f, solicitado %.2f",
                saldoActual, cantidadSolicitada));
        this.saldoActual = saldoActual;
        this.cantidadSolicitada = cantidadSolicitada;
    }

    public double getSaldoActual() { return saldoActual; }
    public double getCantidadSolicitada() { return cantidadSolicitada; }
}

class CuentaBancaria {
    private double saldo;

    public CuentaBancaria(double saldoInicial) {
        this.saldo = saldoInicial;
    }

    public void retirar(double cantidad) {
        if (cantidad > saldo) {
            throw new SaldoInsuficienteException(saldo, cantidad);
        }
        saldo -= cantidad;
        System.out.printf("Retiro: %.2f. Saldo restante: %.2f%n", cantidad, saldo);
    }
}

class EjemploCustomException {
    public static void main(String[] args) {
        CuentaBancaria cuenta = new CuentaBancaria(1000);
        cuenta.retirar(300);

        try {
            cuenta.retirar(800);
        } catch (SaldoInsuficienteException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

Las excepciones personalizadas deben incluir constructores que permitan añadir contexto relevante. Un mensaje descriptivo es esencial para la depuración.

## Stack traces

El *stack trace* (traza de pila) es el registro del camino de llamadas que condujo a la excepción. Se imprime con `printStackTrace()` y contiene la jerarquía de métodos invocados, el archivo y la línea exacta donde se produjo el error.

```java
public class EjemploStackTrace {
    static void nivel3() {
        throw new RuntimeException("Error en nivel 3");
    }

    static void nivel2() {
        nivel3();
    }

    static void nivel1() {
        nivel2();
    }

    public static void main(String[] args) {
        try {
            nivel1();
        } catch (RuntimeException e) {
            System.out.println("Excepcion capturada: " + e.getMessage());
            System.out.println("Stack trace:");
            e.printStackTrace(System.out);
        }
    }
}
```

El stack trace muestra el camino completo desde `main` hasta `nivel3`, incluyendo tipo de excepción, mensaje y ubicación exacta. Para logging en producción, `getMessage()` y `getClass().getSimpleName()` son preferibles a imprimir el stack trace completo.

> En código de producción, el stack trace nunca debe exponerse al usuario final. Se debe registrar internamente y mostrar un mensaje genérico. La clase `Exception` proporciona `getStackTrace()` como array de `StackTraceElement` para análisis programático.
