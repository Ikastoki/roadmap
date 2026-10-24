# Parametros

Los parametros son el mecanismo mediante el cual un metodo recibe datos externos para procesarlos. Java distingue entre parametros formales (los declarados en la definicion del metodo) y argumentos (los valores que se pasan al invocarlo). La comprension del modelo de paso de parametros en Java es esencial para escribir codigo correcto y predecible.

## Parametros formales vs argumentos

Un parametro formal es la variable declarada en la definicion del metodo. Un argumento es el valor concreto que se pasa al invocar el metodo.

```java
// a y b son parametros formales
public int sumar(int a, int b) {
    return a + b;
}

// 5 y 3 son argumentos
int resultado = sumar(5, 3);
```

Los parametros formales actuan como variables locales dentro del metodo, inicializadas con los valores de los argumentos en el momento de la invocacion. Su ambito (scope) es el cuerpo completo del metodo.

> El numero, tipo y orden de los argumentos debe coincidir exactamente con la declaracion de los parametros formales. Java no admite argumentos con nombre como otros lenguajes.

## Paso por valor en Java

Java utiliza exclusivamente paso por valor (pass-by-value). Cuando se invoca un metodo, se crea una copia del argumento y se asigna al parametro formal. Esto significa que el metodo nunca puede modificar la variable original que se paso como argumento.

```java
public void modificar(int x) {
    x = 100; // solo modifica la copia local
}

int numero = 5;
modificar(numero);
System.out.println(numero); // 5, el original no cambia
```

Este comportamiento es igual para todos los tipos de datos en Java. No existe el paso por referencia como en C++ o C#. La confusion surge cuando se trabaja con objetos.

## Paso de primitivos

Al pasar un primitivo, se copia el valor numerico o logico. Cualquier modificacion dentro del metodo queda aislada en la copia.

```java
public void incrementar(int valor) {
    valor++;
}

int contador = 10;
incrementar(contador);
System.out.println(contador); // 10
```

Si se necesita modificar un primitivo externamente, debe devolverse el nuevo valor o encapsularlo en un objeto mutable (como `AtomicInteger` o un array de un elemento).

## Paso de referencias a objetos

Cuando se pasa un objeto, lo que se copia es la referencia (la direccion de memoria), no el objeto mismo. El metodo no puede hacer que la variable original apunte a un objeto distinto, pero si puede modificar el estado interno del objeto referenciado.

```java
public void agregarElemento(List<String> lista) {
    lista.add("nuevo");        // modifica el objeto original
    lista = new ArrayList<>(); // solo afecta a la copia local
}

List<String> datos = new ArrayList<>();
agregarElemento(datos);
System.out.println(datos.size()); // 1, la modificacion persistio
```

La linea `lista = new ArrayList<>()` dentro del metodo no afecta a la variable `datos` del invocante. Solo cambia a que objeto apunta la copia local del parametro.

```java
public void intercambiar(String a, String b) {
    String temp = a;
    a = b;
    b = temp; // solo intercambia las copias locales
}

String x = "primero";
String y = "segundo";
intercambiar(x, y);
System.out.println(x + " " + y); // "primero segundo" (sin cambio)
```

## Varargs

Los varargs (variable-length arguments) permiten pasar un numero variable de argumentos a un metodo. Se declaran con el tipo seguido de `...` y se manejan internamente como un array.

```java
public int sumarTodos(int... numeros) {
    int suma = 0;
    for (int n : numeros) {
        suma += n;
    }
    return suma;
}

// Distintas invocaciones validas:
int s1 = sumarTodos(1, 2, 3);          // 6
int s2 = sumarTodos(10, 20);           // 30
int s3 = sumarTodos();                 // 0
int s4 = sumarTodos(new int[]{1,2,3}); // tambien acepta array directamente
```

El parametro varargs debe ser el ultimo en la lista de parametros. Solo puede haber un varargs por metodo.

```java
public void metodo(String nombre, int... valores) { } // correcto
// public void metodo(int... valores, String nombre) { } // error de compilacion
```

Internamente, el compilador transforma los varargs en arrays. Las llamadas `sumarTodos(1, 2, 3)` se compilan como `sumarTodos(new int[]{1, 2, 3})`.

## Parametros final

Declarar un parametro como `final` impide reasignar el parametro dentro del metodo. Es una practica defensiva que comunica intencion y puede evitar errores accidentales.

```java
public void procesar(final int valor, final String texto) {
    // valor = 10; // error de compilacion
    // texto = "otro"; // error de compilacion
    System.out.println(valor + " " + texto);
}
```

Para parametros de tipo referencia, `final` impide reasignar la referencia, pero no impide modificar el estado interno del objeto:

```java
public void modificarLista(final List<String> lista) {
    lista.add("ok");         // permitido: modifica el objeto
    // lista = new ArrayList<>(); // error: no se puede reasignar
}
```

El uso de `final` en parametros es especialmente util en clases internas anonimas o lambdas, donde las variables externas usadas dentro deben ser effectively final (Java 8+) o estar declaradas como `final`.
