# Variables

Una variable en Java es un espacio de memoria con un nombre, un tipo y un valor que puede cambiar durante la ejecucion del programa. A diferencia de las constantes, las variables permiten almacenar datos que se modifican segun la logica del algoritmo. Comprender los distintos tipos de variables, su alcance y sus reglas de inicializacion es fundamental para escribir codigo correcto y eficiente.

## Declaracion e inicializacion

Declarar una variable consiste en indicar su tipo y nombre al compilador. Inicializarla es asignarle un valor por primera vez. Java exige que toda variable este inicializada antes de ser utilizada; el compilador detecta y rechaza el uso de variables no inicializadas.

```java
// Declaracion
int contador;
String nombre;

// Inicializacion
contador = 0;
nombre = "Ana";

// Declaracion e inicializacion simultanea
int edad = 25;
double salario = 32500.50;

// Multiples declaraciones de la misma tipo
int x, y, z;
x = 1; y = 2; z = 3;
```

> El compilador de Java realiza un analisis de flujo de datos (flow analysis) para determinar si una variable ha sido inicializada en todas las rutas posibles antes de su uso. Esto permite codigo como el siguiente, donde la inicializacion ocurre condicionalmente.

```java
int valor;
if (condicion) {
    valor = 10;
} else {
    valor = 20;
}
// 'valor' esta inicializada en ambas ramas, su uso es valido
System.out.println(valor);
```

Sin embargo, si existe cualquier camino de ejecucion donde la variable no se inicializa, el compilador genera un error:

```java
int valor;
if (condicion) {
    valor = 10;
}
// Error: la variable 'valor' podria no haber sido inicializada
System.out.println(valor);
```

## Tipos de variables

Java distingue tres categorias de variables segun donde se declaran y como se gestionan en memoria:

### Variables locales

Son las que se declaran dentro de un metodo, constructor o bloque. Solo existen mientras se esta ejecutando el bloque donde se declaran y desaparecen cuando sale de el. No tienen valor por defecto; deben inicializarse obligatoriamente antes de usarse.

```java
public void calcular() {
    int resultado = 0;  // variable local
    String mensaje;     // declarada pero no inicializada

    resultado = 5 + 3;
    // System.out.println(mensaje); // ERROR: no inicializada

    mensaje = "calculado";
    System.out.println(mensaje);  // valido
}
```

### Variables de instancia

Son las que se declaran como campos de una clase, fuera de cualquier metodo. Cada objeto creado a partir de la clase tiene su propia copia de estas variables. Adquieren valores por defecto cuando se crea el objeto y mantienen su estado mientras el objeto exista.

```java
public class Cuenta {
    // Variables de instancia
    String titular;
    double saldo;
    boolean activa;

    // Cada objeto Cuenta tiene su propia copia de estos campos
}
```

```java
Cuenta cuenta1 = new Cuenta();
Cuenta cuenta2 = new Cuenta();
// cuenta1.saldo y cuenta2.saldo son variables independientes
```

### Variables de clase (static)

Son campos declarados con el modificador `static`. Existen una unica vez por clase, compartidas por todas las instancias. Se almacenan en la zona de metaspace (antes Java 8, en el PermGen del Method Area) y se inicializan cuando la JVM carga la clase, no cuando se crea un objeto.

```java
public class Contador {
    // Variable de clase: compartida por todas las instancias
    static int totalInstancias = 0;
    // Variable de instancia: unica por objeto
    int id;

    public Contador() {
        totalInstancias++;
        id = totalInstancias;
    }
}
```

```java
Contador c1 = new Contador();
Contador c2 = new Contador();
System.out.println(Contador.totalInstancias); // 2
System.out.println(c1.id); // 1
System.out.println(c2.id); // 2
```

## Alcance (scope)

El alcance de una variable determina donde puede ser referenciada en el codigo. En Java, el alcance se define por el bloque mas interno donde la variable fue declarada.

```java
public void ejemplo() {
    int a = 1;  // alcance: todo el metodo

    if (a > 0) {
        int b = 2;  // alcance: solo dentro de este bloque if
        System.out.println(a + b);  // valido
    }

    // System.out.println(b); // ERROR: 'b' no existe aqui

    for (int i = 0; i < 5; i++) {
        int c = i * 2;  // alcance: cada iteracion del bucle
        System.out.println(c);
    }

    // System.out.println(i); // ERROR: 'i' no existe fuera del for
    // System.out.println(c); // ERROR: 'c' tampoco
}
```

Las variables de ambito reducido se crean y destruyen con mayor frecuencia, lo que ayuda al garbage collector a gestionar la memoria. Tambien reducen la posibilidad de errores al limitar donde puede modificarse una variable.

## Inicializacion por defecto

Las variables de instancia y de clase reciben valores por defecto cuando se crea un objeto o se carga la clase. Las variables locales no tienen valor por defecto y deben inicializarse explicitamente.

| Tipo | Valor por defecto |
|------|-------------------|
| byte, short, int, long | 0 |
| float, double | 0.0 |
| char | '\u0000' (caracter nulo) |
| boolean | false |
| Objectos (String, arrays, etc.) | null |

```java
public class ValoresPorDefecto {
    int entero;        // 0
    double decimal;    // 0.0
    boolean flag;      // false
    String texto;      // null
    int[] arreglo;     // null
}
```

> El valor por defecto de `char` no es el caracter vacio visible, sino el caracter Unicode nulo (`\u0000`), que es un valor numerico 0 invisible pero que ocupan un byte en memoria.

## Efecto final

El termino "efecto final" en el contexto de variables se refiere al concepto de que una variable local puede ser capturada por una lambda o anonymous class solo si es efectivamente final, es decir, si una vez asignada nunca cambia de valor. Desde Java 8, no es necesario declararla como `final` explicitamente, pero el compilador verifica que no se modifique despues de su inicializacion.

```java
int factor = 10;  // no declarada como final, pero nunca se modifica

// Java 8+: esta lambda compila correctamente
Runnable tarea = () -> System.out.println(factor * 5);

// Si se modifica despues, la lambda no compila
// factor = 20;  // esto haria que la lambda anterior fallara
```

Este comportamiento es esencial para la correcta ejecucion de lambdas en hilos distintos al original, ya que la JVM necesita garantizar que el valor capturado no puede cambiar mientras la lambda se ejecuta en otro contexto.
