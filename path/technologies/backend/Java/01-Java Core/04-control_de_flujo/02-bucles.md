# Bucles

Los bucles permiten repetir la ejecución de un bloque de código mientras se cumpla una condición o hasta agotar un conjunto de elementos. Java ofrece tres construcciones de bucle principales: `for`, `while` y `do-while`, cada una con un caso de uso idóneo según se conozca de antemano el número de iteraciones o no.

## Bucle for clasico

El bucle `for` clásico se compone de tres partes: inicialización, condición y actualización, separadas por punto y coma dentro de los paréntesis. Se ejecuta mientras la condición sea `true`.

```java
public class EjemploFor {
    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            System.out.println("Iteracion: " + i);
        }

        // Recorriendo un array
        String[] frutas = {"Manzana", "Pera", "Naranja"};
        for (int i = 0; i < frutas.length; i++) {
            System.out.println(frutas[i]);
        }
    }
}
```

El orden de ejecución es: inicialización → condición → cuerpo → actualización → condición → y así sucesivamente. Las tres partes son opcionales; un `for (;;)` crea un bucle infinito.

## Enhanced for-each

El bucle for-each (introducido en Java 5) simplifica la iteración sobre arrays y colecciones. Elimina la gestión manual del índice y reduce la posibilidad de errores de rango.

```java
public class EjemploForEach {
    public static void main(String[] args) {
        int[] numeros = {10, 20, 30, 40, 50};

        for (int n : numeros) {
            System.out.println("Valor: " + n);
        }

        String[] ciudades = {"Madrid", "Barcelona", "Valencia"};
        for (String ciudad : ciudades) {
            System.out.println(ciudad);
        }
    }
}
```

> El bucle for-each no proporciona acceso al índice. Si se necesita la posición, se debe usar el `for` clásico o recurrir a `IntStream.range()`.

## Bucle while

El bucle `while` evalúa la condición antes de cada iteración. Se usa cuando el número de repeticiones no es conocido de antemano y depende de una condición que se evalúa en tiempo de ejecución.

```java
public class EjemploWhile {
    public static void main(String[] args) {
        int contador = 0;
        while (contador < 5) {
            System.out.println("Contador: " + contador);
            contador++;
        }

        // Simulando un proceso indefinido
        boolean seguir = true;
        int intentos = 0;
        while (seguir) {
            intentos++;
            System.out.println("Intento " + intentos);
            if (intentos >= 3) {
                seguir = false;
            }
        }
    }
}
```

Si la condición nunca se vuelve `false`, se genera un bucle infinito. Es fundamental asegurar que las variables afectadas por la condición se actualicen dentro del cuerpo del bucle.

## Bucle do-while

El bucle `do-while` ejecuta el cuerpo al menos una vez antes de evaluar la condición. Resulta útil cuando se necesita garantizar una ejecución mínima, como en menús interactivos o validación de entrada.

```java
import java.util.Scanner;

public class EjemploDoWhile {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int opcion;

        do {
            System.out.println("=== Menu ===");
            System.out.println("1. Opcion A");
            System.out.println("2. Opcion B");
            System.out.println("0. Salir");
            System.out.print("Elige: ");
            opcion = sc.nextInt();

            switch (opcion) {
                case 1 -> System.out.println("Seleccionaste A");
                case 2 -> System.out.println("Seleccionaste B");
                case 0 -> System.out.println("Saliendo...");
                default -> System.out.println("Opcion no valida");
            }
        } while (opcion != 0);

        sc.close();
    }
}
```

La diferencia respecto a `while` es que el cuerpo se ejecuta siempre antes de la primera comprobación. Si la condición inicial fuera `false`, un `while` no ejecutaría nada, mientras que `do-while` habría ejecutado una vez.

## Bucles infinitos

Un bucle infinito se crea cuando la condición nunca se evalúa como `false`. Esto es intencional en servidores, bucles de eventos y procesos que deben ejecutarse indefinidamente.

```java
public class EjemploInfinito {
    public static void main(String[] args) {
        // for infinito
        for (;;) {
            System.out.println("Ejecutando...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                break;
            }
        }

        // while infinito
        while (true) {
            System.out.println("Procesando...");
            break; // Salir con break
        }
    }
}
```

> Los bucles infinitos en aplicaciones de servidor son comunes. En programas de consola, se debe incluir siempre un mecanismo de salida explícito (`break` o `return`).

## Anidamiento de bucles

Los bucles pueden anidarse para procesar estructuras bidimensionales, combinar iteraciones, o implementar algoritmos como búsqueda exhaustiva.

```java
public class EjemploAnidamiento {
    public static void main(String[] args) {
        // Tabla de multiplicar
        for (int i = 1; i <= 5; i++) {
            for (int j = 1; j <= 5; j++) {
                System.out.printf("%4d", i * j);
            }
            System.out.println();
        }

        // Busqueda en matriz
        int[][] matriz = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
        int objetivo = 5;
        boolean encontrado = false;

        for (int i = 0; i < matriz.length && !encontrado; i++) {
            for (int j = 0; j < matriz[i].length && !encontrado; j++) {
                if (matriz[i][j] == objetivo) {
                    System.out.printf("Encontrado en [%d][%d]%n", i, j);
                    encontrado = true;
                }
            }
        }
    }
}
```

La condición `!encontrado` en el `for` permite detener la búsqueda en cuanto se localiza el elemento, evitando iteraciones innecesarias.

## Cuando usar cada tipo

| Bucle | Cuándo usarlo |
|---|---|
| `for` | Número de iteraciones conocido de antemano |
| `for-each` | Recorrer arrays o colecciones sin necesidad del índice |
| `while` | Iterar mientras una condición sea verdadera, sin conocer el número de veces |
| `do-while` | Garantizar al menos una ejecución antes de comprobar la condición |

La elección incorrecta no genera errores de compilación, pero sí código menos legible y más propenso a fallos lógicos. Si se recorre un `ArrayList` con un `for` clásico accediendo a `get(i)`, un `for-each` sería más idiomático. Si se implementa un menú de consola con `while`, se perdería la primera ejecución que `do-while` garantiza.
