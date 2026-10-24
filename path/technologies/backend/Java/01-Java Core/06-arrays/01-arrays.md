# Arrays

Un array en Java es un objeto contenedor que almacena un numero fijo de elementos del mismo tipo. Los arrays son la estructura de datos mas basica y eficiente para trabajar con colecciones de datos, y constituyen la base sobre la que se construyen las colecciones de alto nivel como `ArrayList` o `HashMap`.

## Declaracion

La declaracion de un array especifica el tipo de sus elementos y unos corchetes. Los corchetes pueden colocarse tras el tipo o tras el nombre de la variable. La convencion predominante en Java es colocarlos tras el tipo.

```java
int[] numeros;      // forma preferida
String nombres[];   // valido pero menos usado
double[] valores;
```

En este punto, `numeros` es una variable de tipo `int[]` (array de enteros), pero el array aun no existe. Solo se ha declarado la referencia, que inicialmente vale `null`.

## Instanciacion

Para crear el array se usa el operador `new` seguido del tipo y el tamaño entre corchetes. El tamaño debe ser un entero no negativo.

```java
int[] numeros = new int[5];       // array de 5 enteros
String[] nombres = new String[3];  // array de 3 referencias a String
```

Cuando se crea un array con `new`, todos sus elementos se inicializan automaticamente con el valor por defecto del tipo: `0` para numericos, `false` para `boolean`, `'\u0000'` para `char`, y `null` para tipos de referencia.

Tambien es posible instanciar e inicializar en un solo paso con la sintaxis de initializer list:

```java
int[] numeros = {10, 20, 30, 40, 50};
String[] colores = {"rojo", "verde", "azul"};
```

Esta sintaxis solo es valida en la declaracion. Para reasignar un array existente debe usarse `new` con initializer list:

```java
numeros = new int[]{10, 20, 30}; // valido
// numeros = {10, 20, 30}; // ERROR
```

## Acceso por indice

Los elementos se acceden mediante corchetes con el indice, que comienza en 0. El ultimo elemento tiene indice `length - 1`.

```java
int[] numeros = {10, 20, 30, 40, 50};

System.out.println(numeros[0]);  // 10
System.out.println(numeros[2]);  // 30
System.out.println(numeros[4]);  // 50

numeros[1] = 25; // modifica el segundo elemento
```

Acceder a un indice negativo o mayor o igual que `length` produce un `ArrayIndexOutOfBoundsException` en tiempo de ejecucion.

```java
int[] arr = new int[3];
// System.out.println(arr[3]); // ArrayIndexOutOfBoundsException
// System.out.println(arr[-1]); // ArrayIndexOutOfBoundsException
```

## La propiedad length

Todo array tiene una propiedad `length` (no es un metodo) que indica el numero de elementos que puede contener.

```java
int[] numeros = {10, 20, 30};
System.out.println(numeros.length); // 3
```

`length` es un campo `public final` del objeto array. No debe confundirse con `String.length()` (que es un metodo). La ausencia de parentesis es el indicador visual.

## Recorrido con for tradicional

El bucle `for` clasico con indice permite recorrer el array y, si es necesario, modificar elementos o conocer la posicion.

```java
int[] numeros = {10, 20, 30, 40, 50};

// Sumar todos los elementos
int suma = 0;
for (int i = 0; i < numeros.length; i++) {
    suma += numeros[i];
}

// Modificar elementos (duplicar cada valor)
for (int i = 0; i < numeros.length; i++) {
    numeros[i] *= 2;
}
```

## Recorrido con for-each

El bucle `for-each` (enhanced for loop) recorre los elementos sin necesidad de indice. Es mas legible pero no permite modificar el array ni conocer la posicion del elemento.

```java
int[] numeros = {10, 20, 30};

for (int n : numeros) {
    System.out.println(n);
}

// Calcular maximo
int max = Integer.MIN_VALUE;
for (int n : numeros) {
    if (n > max) max = n;
}
```

> Intentar modificar la variable del for-each (`n`) no afecta al array. `n` es una copia del valor en cada iteracion. Para modificar elementos debe usarse el bucle con indice.

## Arrays como parametros y retorno

Los arrays pueden pasarse como argumentos y devolverse como resultado de metodos. Al ser objetos en Java, se pasan por valor de referencia.

```java
public class UtilArrays {

    // Recibe un array y modifica su contenido
    public static void rellenar(int[] arr, int valor) {
        for (int i = 0; i < arr.length; i++) {
            arr[i] = valor;
        }
    }

    // Devuelve un nuevo array
    public static int[] clonar(int[] original) {
        int[] copia = new int[original.length];
        for (int i = 0; i < original.length; i++) {
            copia[i] = original[i];
        }
        return copia;
    }

    // Varargs: se recibe como array
    public static int sumarTodos(int... numeros) {
        int total = 0;
        for (int n : numeros) {
            total += n;
        }
        return total;
    }
}

// Uso:
int[] datos = {1, 2, 3};
UtilArrays.rellenar(datos, 0);     // datos queda {0, 0, 0}
int[] copia = UtilArrays.clonar(datos);

int suma = UtilArrays.sumarTodos(5, 10, 15); // 30
```

Al pasar un array como parametro, el metodo recibe la referencia. Las modificaciones internas afectan al array original. Para proteger el array original, debe pasarse una copia defensiva con `clone()` o `Arrays.copyOf()`.
