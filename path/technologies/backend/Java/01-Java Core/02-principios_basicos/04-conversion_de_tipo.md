# Conversion de tipo

En Java, toda variable tiene un tipo fijo en tiempo de compilacion, pero los valores pueden transformarse de un tipo a otro mediante conversiones. El lenguaje distingue entre conversiones implicitas (widening) y explicitas (narrowing), ademas de mecanismos automaticos como el autoboxing/unboxing y las conversiones con cadenas.

## Widening (conversion implicita)

La conversion widening, tambien llamada promocion de tipo, ocurre automaticamente cuando se asigna un valor de un tipo de menor rango a otro de mayor rango. No hay perdida de informacion porque el destino tiene suficiente capacidad.

```
byte -> short -> int -> long -> float -> double
               ^
              char (tambien promociona a int)
```

```java
byte b = 100;
short s = b;    // byte -> short, implicito
int i = s;      // short -> int, implicito
long l = i;     // int -> long, implicito
float f = l;    // long -> float, implicito (posible perdida de precision)
double d = f;   // float -> double, implicito
```

Aunque `long` tiene 64 bits y `float` solo 32, la conversion de `long` a `float` es widening porque `float` puede representar valores mas grandes en magnitud, aunque con menor precision. El compilador acepta la conversion sin requerir cast explicito.

## Narrowing (conversion explicita con casting)

La conversion narrowing ocurre cuando se asigna un valor de un tipo de mayor rango a uno de menor rango. Java no permite estas conversiones de forma implicita porque existe riesgo de perdida de datos. El programador debe usar un operador de casting.

```java
double d = 3.14159;
int i = (int) d;        // 3, se trunca la parte decimal

int grande = 300;
byte b = (byte) grande; // 44, se pierden bits significativos

long l = 123456789L;
int i2 = (int) l;       // posible truncamiento si excede Integer.MAX_VALUE
```

El casting no modifica la variable original; produce una nueva representacion del valor en el tipo destino. Si el valor original excede el rango del tipo destino, los bits mas significativos simplemente se descartan.

> Al hacer un cast entre tipos enteros, Java descarta los bits de mayor orden. Por ejemplo, `(byte) 300` da 44 porque 300 en binario es `0001 0010 1100`, y al truncar a 8 bits queda `0010 1100`, que es 44.

## Autoboxing y unboxing

Desde Java 5, la conversion entre tipos primitivos y sus envoltorios (wrappers) es automatica.

```java
Integer obj = 42;        // autoboxing: int -> Integer
int valor = obj;         // unboxing: Integer -> int

List<Integer> numeros = new ArrayList<>();
numeros.add(10);         // autoboxing en el argumento
int suma = numeros.get(0) + 5; // unboxing + autoboxing
```

El autoboxing utiliza los metodos `valueOf()` de cada wrapper, que implementan un cache para valores comunes (-128 a 127 para `Integer`, booleanos y algunos rangos de `Byte`, `Short`, `Long`). Esto significa que dos instancias de `Integer.valueOf(42)` pueden ser la misma referencia.

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (por cache)

Integer c = 200;
Integer d = 200;
System.out.println(c == d);  // false (fuera del cache)
```

> El autoboxing/unboxing es comodo pero puede ocultar creaciones innecesarias de objetos. En bucles largos con primitivos, usar el tipo primitivo directamente es mas eficiente.

## Conversion con Strings

La conversion entre tipos primitivos y `String` no es automatica; requiere metodos explicitos.

```java
int numero = 456;
String texto = String.valueOf(numero);     // "456"
String binario = Integer.toBinaryString(10); // "1010"
String hex = Integer.toHexString(255);      // "ff"

String entrada = "123";
int valor = Integer.parseInt(entrada);     // 123
double d = Double.parseDouble("3.14");     // 3.14
boolean b = Boolean.parseBoolean("true");  // true
```

`Integer.parseInt()` lanza `NumberFormatException` si la cadena no contiene un numero valido. Es responsabilidad del programador manejar esta excepción o validar la entrada previamente.

La concatenacion con `+` entre un `String` y un primitivo convierte automaticamente el primitivo a `String` mediante el mecanismo de string conversion del compilador:

```java
String mensaje = "El valor es " + 42; // "El valor es 42"
```

## Perdida de datos en las conversiones

Cada conversion narrowing puede provocar perdida de informacion. Los casos tipicos son:

**Truncamiento de parte decimal:** al convertir `double` o `float` a entero, la parte fraccionaria se descarta sin redondeo.

**Desbordamiento de rango:** al asignar un valor que excede el maximo del tipo destino, los bits de mayor peso se pierden.

```java
int max = Integer.MAX_VALUE; // 2147483647
int min = Integer.MIN_VALUE; // -2147483648

int desbordado = max + 1;   // -2147483648 (desbordamiento silencioso)
```

Java no notifica el desbordamiento aritmetico en tipos enteros. La unica proteccion nativa es el desbordamiento de `float` y `double`, que produce valores especiales como `Infinity` o `-Infinity`. Para operaciones donde el desbordamiento es critico, deben usarse metodos como `Math.addExact()` que lanzan `ArithmeticException`.
