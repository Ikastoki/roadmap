# Math

La clase `java.lang.Math` proporciona metodos estaticos para operaciones matematicas elementales. No se instancia -- su constructor es privado -- y todos sus metodos son `static`. Opera exclusivamente sobre tipos primitivos (`double`, `float`, `int`, `long`) y sus constantes principales son `E` y `PI`.

## Constantes

```java
double pi = Math.PI;      // 3.141592653589793
double e = Math.E;        // 2.718281828459045
```

## Metodos de valor absoluto

```java
int absInt = Math.abs(-10);     // 10
double absD = Math.abs(-3.5);   // 3.5
long absL = Math.abs(-100L);    // 100L
```

> `Math.abs(Integer.MIN_VALUE)` devuelve `Integer.MIN_VALUE` (negativo) porque no existe representacion positiva para ese valor en complemento a dos. Lo mismo ocurre con `Long.MIN_VALUE`.

## Metodos de comparacion

```java
int max = Math.max(10, 20);         // 20
int min = Math.min(10, 20);         // 10
float maxF = Math.max(3.14f, 2.5f); // 3.14f
```

No existe `Math.clamp()` hasta Java 21. En versiones anteriores hay que implementarlo manualmente:

```java
int clamp(int valor, int min, int max) {
    return Math.max(min, Math.min(max, valor));
}
```

## Metodos de redondeo

```java
long redondeado = Math.round(3.6);   // 4
int redondeadoF = Math.round(3.6f);  // 4
double ceil = Math.ceil(3.2);        // 4.0
double floor = Math.floor(3.8);      // 3.0
double rint = Math.rint(3.5);        // 4.0 (redondeo al entero par mas cercano)
```

`Math.round()` usa la regla de redondeo "redondear hacia arriba" a partir de .5. `Math.rint()` redondea al entero mas cercano, y en casos equidistantes (.5) redondea al entero par.

## Exponenciales y logaritmos

```java
double raiz = Math.sqrt(16);         // 4.0
double cubo = Math.cbrt(27);         // 3.0
double potencia = Math.pow(2, 10);   // 1024.0
double exp = Math.exp(1);            // E^1 = 2.718...
double log = Math.log(10);           // logaritmo natural de 10
double log10 = Math.log10(1000);     // 3.0
double log1p = Math.log1p(0);        // log(1+0) = 0, precision para x pequeno
double expm1 = Math.expm1(1);        // e^1 - 1
```

`Math.log1p(x)` y `Math.expm1(x)` ofrecen mayor precision numerica que `Math.log(1+x)` y `Math.exp(x)-1` cuando `x` es muy pequeno.

## Trigonometria

```java
double sin = Math.sin(Math.PI / 2);   // 1.0
double cos = Math.cos(0);             // 1.0
double tan = Math.tan(0);             // 0.0

double asin = Math.asin(1);           // PI/2
double acos = Math.acos(1);           // 0.0
double atan = Math.atan(1);           // PI/4
double atan2 = Math.atan2(1, 1);      // PI/4

double grados = Math.toDegrees(Math.PI); // 180.0
double rads = Math.toRadians(180);       // PI
```

`Math.atan2(y, x)` es preferible a `Math.atan(y/x)` porque maneja correctamente los cuadrantes y evita division por cero.

## Aleatorios

```java
double r = Math.random(); // [0.0, 1.0)
```

`Math.random()` usa internamente `java.util.Random`. Para generacion de numeros aleatorios mas controlada (semilla, distribucion, concurrencia) deben usarse `java.util.Random`, `java.util.concurrent.ThreadLocalRandom`, o `java.security.SecureRandom`.

Generar enteros en un rango:

```java
int aleatorio = (int)(Math.random() * 100); // [0, 99]
```

## Metodos adicionales

```java
double signo = Math.signum(-5.5);      // -1.0 (devuelve el signo)
float copySign = Math.copySign(3.5f, -2.0f); // -3.5f
double ulp = Math.ulp(1.0);            // unidad de ultima precision
double nextAfter = Math.nextAfter(1.0, 0); // 0.9999999999999999
int floorDiv = Math.floorDiv(-7, 3);   // -3 (division con redondeo hacia abajo)
int floorMod = Math.floorMod(-7, 3);   // 2
```

`Math.floorDiv()` y `Math.floorMod()` implementan division y modulo donde el resultado siempre tiene el signo del divisor, a diferencia de la division entera nativa de Java que trunca hacia cero.

## Precision y rendimiento

`Math` delega en `StrictMath` para la mayoria de operaciones. `StrictMath` garantiza resultados identicos en todas las plataformas (puerto de la biblioteca fdlibm). `Math` permite que la JVM use implementaciones nativas optimizadas (como x87 o SSE) que pueden diferir ligeramente entre plataformas pero ofrecen mejor rendimiento.

```java
// StrictMath garantiza portabilidad
double r1 = StrictMath.sin(x);
// Math permite optimizaciones nativas (por defecto)
double r2 = Math.sin(x);
```
