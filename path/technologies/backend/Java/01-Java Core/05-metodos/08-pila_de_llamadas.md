# Pila de llamadas

La pila de llamadas (call stack) es una estructura de datos que la JVM utiliza para gestionar la ejecucion de metodos. Cada vez que se invoca un metodo, la JVM crea un nuevo marco (stack frame) que se apila sobre el anterior. Cuando el metodo termina, su marco se desapila y el control regresa al metodo invocante.

## Como se apilan las llamadas

Cada hilo de ejecucion (thread) tiene su propia pila. Cuando un metodo invoca a otro, la JVM reserva un nuevo frame que contiene los parametros, las variables locales del metodo, y la direccion de retorno.

```java
public class PilaEjemplo {

    static void metodoA() {
        System.out.println("Entra en A");
        metodoB();
        System.out.println("Sale de A");
    }

    static void metodoB() {
        System.out.println("Entra en B");
        metodoC();
        System.out.println("Sale de B");
    }

    static void metodoC() {
        System.out.println("Entra en C");
        System.out.println("Sale de C");
    }

    public static void main(String[] args) {
        metodoA();
    }
}
```

La salida muestra el orden de ejecucion:

```
Entra en A
Entra en B
Entra en C
Sale de C
Sale de B
Sale de A
```

La pila crece hacia abajo (push) con cada llamada y se contrae (pop) al retornar. El metodo actual esta siempre en la cima de la pila. Si se imprime el stack trace en ese momento, se obtiene la secuencia completa de llamadas que llevaron hasta el punto actual.

## StackOverflowError

El `StackOverflowError` ocurre cuando la pila se queda sin espacio. La causa tipica es una recursion infinita o una recursion demasiado profunda.

```java
public class StackOverflowEjemplo {

    static void recursivo() {
        recursivo(); // nunca termina
    }

    public static void main(String[] args) {
        recursivo(); // lanza StackOverflowError
    }
}
```

Cada llamada recursiva consume un frame en la pila. Cuando se alcanza el limite (tipicamente entre 8 KB y 1 MB dependiendo de la configuracion de la JVM y el sistema operativo), la JVM lanza `StackOverflowError` y el hilo muere.

```java
public class RecursionControlada {

    static int llamados = 0;

    static void recursivo() {
        llamados++;
        if (llamados % 1000 == 0) {
            System.out.println("Llamada " + llamados);
        }
        recursivo();
    }

    public static void main(String[] args) {
        try {
            recursivo();
        } catch (StackOverflowError e) {
            System.out.println("Desbordamiento en llamada " + llamados);
        }
    }
}
```

En una maquina tipica, el `StackOverflowError` suele aparecer entre 10,000 y 50,000 llamadas recursivas. Capturar `StackOverflowError` no es recomendable en produccion, pero puede ser util para diagnosticar el limite en desarrollo.

> `StackOverflowError` es un `Error`, no una `Exception`. Pertenece a la categoria de condiciones anormales de las que una aplicacion no deberia intentar recuperarse. Es mejor prevenirla limitando la profundidad recursiva o transformando la recursion en iteracion.

## StackTraceElement

Cada elemento del stack trace es un objeto `StackTraceElement` que contiene informacion sobre el punto de ejecucion: clase, metodo, archivo fuente, y numero de linea.

```java
public class StackTraceEjemplo {

    public static void main(String[] args) {
        metodoX();
    }

    static void metodoX() {
        metodoY();
    }

    static void metodoY() {
        StackTraceElement[] stack = Thread.currentThread().getStackTrace();
        for (StackTraceElement e : stack) {
            System.out.printf("%s.%s(%s:%d)%n",
                e.getClassName(),
                e.getMethodName(),
                e.getFileName(),
                e.getLineNumber());
        }
    }
}
```

Salida tipica:

```
java.base/java.lang.Thread.getStackTrace(Thread.java:1610)
StackTrazabilidad.metodoY(StackTrazabilidad.java:18)
StackTrazabilidad.metodoX(StackTrazabilidad.java:13)
StackTrazabilidad.main(StackTrazabilidad.java:9)
```

El primer elemento (indice 0) es la llamada a `getStackTrace` misma. El segundo es el metodo actual. A partir de ahi, los elementos siguientes muestran la cadena de invocaciones hasta el metodo `main`.

## Analizar y depurar stack traces

El stack trace es la herramienta principal para diagnosticar excepciones. Al leerlo, se sigue desde la cima (donde ocurrio la excepcion) hacia abajo (las llamadas previas).

```
Exception in thread "main" java.lang.NullPointerException:
    at com.ejemplo.Servicio.procesar(Servicio.java:25)
    at com.ejemplo.Controlador.ejecutar(Controlador.java:42)
    at com.ejemplo.Main.main(Main.java:10)
```

Para depurar eficazmente un stack trace:

1. Leer el tipo de excepcion y el mensaje (`NullPointerException`)
2. Localizar la primera linea que hace referencia a codigo propio (no de librerias)
3. Revisar la linea indicada (`Servicio.java:25`) para entender que objeto era `null`
4. Si es necesario, seguir la cadena hacia atras para entender el flujo de llamadas

Los IDE modernos integran la navegacion desde el stack trace: haciendo clic en cada linea se abre el archivo fuente en la posicion indicada. Ademas, los logs estructurados y los APM tools registran stack traces completos para su analisis posterior.

La comprension de la pila de llamadas es esencial no solo para depurar errores, sino para entender el rendimiento: pilas muy profundas incrementan la presion sobre el GC y ralentizan la ejecucion. Herramientas como profilers (YourKit, JProfiler) y el propio `jstack` de la JDK permiten inspeccionar la pila de hilos en tiempo real.
