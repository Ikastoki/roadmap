# Continue, break, return

Las sentencias de control de flujo `break`, `continue` y `return` permiten alterar el curso normal de ejecución dentro de bucles, sentencias `switch` y métodos. Cada una tiene un efecto distinto: `break` termina una estructura, `continue` salta a la siguiente iteración, y `return` abandona el método actual.

## Break

La sentencia `break` detiene la ejecución del bucle o `switch` más cercano y transfiere el control a la sentencia siguiente. En un bucle anidado, `break` solo sale del bucle más interno.

```java
public class EjemploBreak {
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            if (i == 5) {
                System.out.println("Encontrado 5, saliendo del bucle");
                break;
            }
            System.out.println("i = " + i);
        }

        System.out.println("Bucle terminado");
    }
}
```

Salida:

```
i = 0
i = 1
i = 2
i = 3
i = 4
Encontrado 5, saliendo del bucle
Bucle terminado
```

En `switch`, `break` es obligatorio para evitar el *fall-through* al caso siguiente. Sin `break`, la ejecución continúa con el caso siguiente independientemente de si la condición coincide.

```java
int valor = 2;
switch (valor) {
    case 1:
        System.out.println("Uno");
        break; // Sin esto, caeria al case 2
    case 2:
        System.out.println("Dos");
        break;
    case 3:
        System.out.println("Tres");
        break;
}
```

## Continue

La sentencia `continue` salta el resto del cuerpo del bucle y pasa directamente a la evaluación de la condición (en `while`/`do-while`) o a la actualización del contador (en `for`). No termina el bucle; simplemente omite la iteración actual.

```java
public class EjemploContinue {
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            if (i % 2 == 0) {
                continue; // Saltar pares
            }
            System.out.println("Impar: " + i);
        }
    }
}
```

Salida:

```
Impar: 1
Impar: 3
Impar: 5
Impar: 7
Impar: 9
```

En un `while`, es crucial que la actualización de la variable de control se realice antes del `continue`, de lo contrario se generaría un bucle infinito.

```java
int i = 0;
while (i < 10) {
    i++; // Actualizar ANTES del continue
    if (i % 2 == 0) {
        continue;
    }
    System.out.println("Impar: " + i);
}
```

## Return

La sentencia `return` termina la ejecución del método actual y devuelve el control al lugar que lo invocó. Si el método declara un tipo de retorno, `return` debe ir seguido de una expresión compatible.

```java
public class EjemploReturn {
    static boolean buscar(int[] arreglo, int objetivo) {
        for (int i = 0; i < arreglo.length; i++) {
            if (arreglo[i] == objetivo) {
                return true; // Salir inmediatamente del metodo
            }
        }
        return false;
    }

    static void saludar(String nombre) {
        if (nombre == null) {
            System.out.println("Nombre no valido");
            return; // Salir sin devolver valor (metodo void)
        }
        System.out.println("Hola, " + nombre);
    }

    public static void main(String[] args) {
        int[] nums = {3, 7, 1, 9, 4};
        System.out.println("Encontrado: " + buscar(nums, 9));
        System.out.println("Encontrado: " + buscar(nums, 5));
        saludar(null);
        saludar("Laura");
    }
}
```

En métodos `void`, `return` sin valor es opcional al final del método, pero puede utilizarse para salir anticipadamente ante condiciones de error o casos especiales.

> `return` en un bucle es una forma eficiente de buscar un valor: tan pronto se encuentra, el método termina sin recorrer el resto del arreglo.

## Break con label

Java permite etiquetar sentencias de bucle con un nombre (label). Al asociar `break` a un label, se sale del bucle etiquetado, no solo del interno. Esto resulta útil en bucles anidados.

```java
public class EjemploBreakLabel {
    public static void main(String[] args) {
        externo:
        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 5; j++) {
                if (i == 2 && j == 3) {
                    System.out.printf("Deteniendo en [%d][%d]%n", i, j);
                    break externo;
                }
                System.out.printf("[%d][%d] ", i, j);
            }
            System.out.println();
        }
        System.out.println("Bucle externo terminado");
    }
}
```

Sin el label, `break` solo saliría del bucle interno (el de `j`). Con `break externo`, se terminan ambos bucles simultáneamente.

> El uso de labels es poco frecuente en código productivo, pero es la herramienta correcta cuando se necesita salir de múltiples niveles de anidamiento sin introducir banderas booleanas.

## Continue con label

`continue` también acepta labels para saltar a la siguiente iteración de un bucle externo.

```java
public class EjemploContinueLabel {
    public static void main(String[] args) {
        externo:
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) {
                if (j == 2) {
                    continue externo; // Saltar siguiente iteracion del bucle externo
                }
                System.out.printf("[%d][%d] ", i, j);
            }
            System.out.println();
        }
    }
}
```

Salida:

```
[0][0] [0][1] [1][0] [1][1] [2][0] [2][1] [3][0] [3][1]
```

Cuando `j == 2`, `continue externo` salta directamente a la siguiente iteración del bucle externo (`i++`), omitiendo las restantes iteraciones del bucle interno.

> Los labels en `continue` son aún menos comunes que en `break`. Si el código requiere `continue externo` frecuentemente, merece la pena reestructurar la lógica en métodos auxiliares.
