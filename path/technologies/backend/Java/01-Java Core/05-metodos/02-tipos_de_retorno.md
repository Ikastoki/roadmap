# Tipos de retorno

Cada metodo en Java declara un tipo de retorno que especifica que valor, si hay alguno, devuelve al invocante. El sistema de tipos de Java permite devolver tanto primitivos como objetos, incluyendo arrays, y restringe mediante verificacion en tiempo de compilacion que todas las rutas de ejecucion entreguen el valor prometido.

## void

Un metodo declarado con `void` no devuelve ningun valor. La sentencia `return` es opcional, y si se usa, aparece sin expresion. Sirve unicamente para salir del metodo antes de alcanzar el final.

```java
public void imprimirMensaje(String msg) {
    if (msg == null) {
        return; // sale del metodo
    }
    System.out.println(msg);
}
```

Aunque `void` no es un tipo en el sentido de que no hay valores de tipo `void`, el compilador trata `void` como tipo de retorno valido para metodos que solo ejecutan efectos secundarios.

## Tipos primitivos

Pueden devolverse cualquiera de los ocho tipos primitivos de Java: `byte`, `short`, `int`, `long`, `float`, `double`, `char`, `boolean`.

```java
public int obtenerEdad() { return 25; }
public boolean esMayorEdad(int edad) { return edad >= 18; }
public double calcularPromedio(double[] valores) {
    double suma = 0;
    for (double v : valores) suma += v;
    return suma / valores.length;
}
```

Si el metodo declara un tipo primitivo como retorno, el valor devuelto debe ser compatible por asignacion con ese tipo. Java realiza las conversiones widening de forma automatica, pero las narrowing requieren casting.

## Tipos de referencia

Cualquier clase o interfaz puede ser tipo de retorno. El metodo devuelve una referencia al objeto, no el objeto en si.

```java
public String obtenerSaludo() {
    return "Hola Mundo";
}

public Usuario buscarUsuario(int id) {
    // logica de busqueda
    return new Usuario(id, "Juan");
}
```

El valor `null` es un retorno valido para cualquier tipo de referencia. Quien invoca el metodo debe estar preparado para manejar `null` y evitar `NullPointerException`.

```java
public Usuario buscar(int id) {
    if (id < 0) return null;
    return baseDatos.findById(id);
}
```

## Objetos como retorno

Los objetos devueltos pueden ser instancias de la clase declarada o de cualquier subclase (polimorfismo de retorno). Tambien pueden ser instancias de clases anonimas o internas.

```java
public List<String> obtenerLista() {
    List<String> lista = new ArrayList<>();
    lista.add("Java");
    lista.add("Python");
    return lista; // ArrayList implementa List
}

public Comparable obtenerComparable() {
    return 42; // Integer implementa Comparable
}
```

## Arrays como retorno

Los arrays son objetos en Java, por lo que pueden devolverse como cualquier otro tipo de referencia.

```java
public int[] generarArray(int tamanio) {
    int[] arr = new int[tamanio];
    for (int i = 0; i < tamanio; i++) {
        arr[i] = i * i;
    }
    return arr;
}

public String[] dividirCadena(String texto) {
    return texto.split(",");
}
```

Al devolver un array, se devuelve la referencia al objeto array. El invocante obtiene acceso directo a los mismos elementos, no una copia. Para evitar mutaciones externas no deseadas, debe devolverse una copia defensiva con `clone()` o `Arrays.copyOf()`.

## Retorno multiple con arrays u objetos

Un metodo solo puede devolver un valor. Para simular retorno multiple se usan arrays, colecciones, o clases contenedoras.

```java
// Retorno multiple con array
public int[] minMax(int[] numeros) {
    int min = Integer.MAX_VALUE;
    int max = Integer.MIN_VALUE;
    for (int n : numeros) {
        if (n < min) min = n;
        if (n > max) max = n;
    }
    return new int[]{min, max};
}

// Retorno multiple con objeto
public class Resultado {
    public final double media;
    public final double desviacion;
    public Resultado(double media, double desviacion) {
        this.media = media;
        this.desviacion = desviacion;
    }
}

public Resultado calcularEstadisticas(double[] datos) {
    double media = calcularMedia(datos);
    double desviacion = calcularDesviacion(datos, media);
    return new Resultado(media, desviacion);
}
```

Java 14 introdujo los `record` como alternativa concisa para este patron:

```java
public record Punto(int x, int y) {}

public Punto origen() {
    return new Punto(0, 0);
}
```

## La sentencia return

`return` termina la ejecucion del metodo y entrega el control al invocante. Si el metodo declara un tipo de retorno distinto de `void`, `return` debe ir acompañado de una expresion del tipo declarado.

```java
public int ejemplo(boolean condicion) {
    if (condicion) {
        return 10;
    }
    // Error de compilacion: falta return si condicion es false
}
```

El compilador exige que todas las rutas de ejecucion posibles terminen con un `return`. Esto incluye bloques `if-else`, `switch`, bucles con condiciones que el compilador no puede probar como siempre verdaderas, y bloques `try-catch`.

```java
public int corregido(boolean condicion) {
    if (condicion) {
        return 10;
    } else {
        return 20;
    }
}
```
