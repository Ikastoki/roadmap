# Tipos de datos

Java es un lenguaje fuertmente tipado, lo que significa que cada variable, expresion y valor tiene un tipo determinado en tiempo de compilacion y el compilador verifica las compatibilidades antes de generar el bytecode. Los tipos de datos en Java se dividen en dos grandes categorias: primitivos y referencia.

## Tipos primitivos

Los tipos primitivos representan valores basicos almacenados directamente en la pila (stack) del metodo, no en el heap. Son inmutables en el sentido de que su valor se almacena tal cual, sin envoltura de objetos.

### byte

Ocupa exactamente 1 byte (8 bits) y representa enteros con signo en el rango de -128 a 127. Se utiliza cuando se necesita optimizar el consumo de memoria en arrays grandes o en protocolos de red donde el ancho de banda es critico.

```java
byte intensidad = 100;
byte temperatura = -40;
// byte grande = 200; // ERROR: 208 excede el rango
```

### short

Ocupa 2 bytes (16 bits) con rango de -32,768 a 32,767. Es poco comun en uso general pero aparece en APIs de bajo nivel y formatos binarios.

```java
short distancia = 15000;
short negativo = -15000;
```

### int

Ocupa 4 bytes (32 bits) con rango de -2^31 a 2^31 - 1 (aproximadamente -2,147,483,648 a 2,147,483,647). Es el tipo entero por defecto en Java y el mas utilizado para contadores, indices y operaciones enteras generales.

```java
int poblacion = 47000000;
int maximo = Integer.MAX_VALUE;  // 2,147,483,647
int minimo = Integer.MIN_VALUE;  // -2,147,483,648
```

### long

Ocupa 8 bytes (64 bits) y se usa cuando el rango de `int` es insuficiente. Se declara con el sufijo `L` o `l` para indicar al compilador que el literal es de tipo long. Sin el sufijo, un literal grande se interpreta como `int` y puede generar un error de compilacion.

```java
long poblacionMundial = 8_000_000_000L;
long timestamp = 1700000000000L;
long maximo = Long.MAX_VALUE;  // 9,223,372,036,854,775,807
```

> Los guiones bajos en literales numericos (`8_000_000_000L`) son solo decorativos y no afectan el valor. Mejoran la legibilidad al separar grupos de digitas, igual que los separadores de miles en escritura convencional.

### float

Ocupa 4 bytes (32 bits) y representa numeros de punto flotante de precision simple (IEEE 754). Se declara con el sufijo `F` o `f`. Es insuficiente para calculos financieros por su precision limitada (aproximadamente 7 digitos significativos).

```java
float precio = 19.99f;
float pi = 3.14f;
```

### double

Ocupa 8 bytes (64 bits) y es el tipo de punto flotante por defecto en Java. Tiene precision doble (aproximadamente 15-16 digitos significativos). Es el tipo recomendado para la mayoria de calculos con decimales.

```double precio = 19.99;
double pi = Math.PI;
double granNumero = 1.7976931348623157E308; // Double.MAX_VALUE
```

### char

Ocupa 2 bytes (16 bits) y representa un unico caracter Unicode. A diferencia de C/C++, el tipo char en Java es sin signo y almacena valores desde 0 hasta 65,535. Soporta caracteres internacionales, emojis y simbolos especiales.

```java
char letra = 'A';
char numeral = '\u0040';  // '@' en Unicode
char euro = '\u20AC';     // '€'
char emoji = '\uD83D\uDE00';  // emoji (surrogate pair)
```

### boolean

No tiene un tamano fijo definido por el estandar; la JVM decide su representacion interna. Solo puede tomar los valores `true` o `false`. No se convierte implicitamente a enteros (a diferencia de C/C++ donde 0 es false y cualquier otro valor es true).

```java
boolean activo = true;
boolean tieneError = false;
```

## Clases wrapper (envoltorios)

Cada tipo primitivo tiene una clase wrapper correspondiente que lo envuelve como un objeto en el heap. Estas clases son necesarias cuando se requiere tratar un valor primitivo como objeto, por ejemplo, en colecciones generics que solo aceptan tipos referencia.

| Primitivo | Wrapper |
|-----------|---------|
| byte | Byte |
| short | Short |
| int | Integer |
| long | Long |
| float | Float |
| double | Double |
| char | Character |
| boolean | Boolean |

```java
// Creacion de objetos wrapper
Integer entero = Integer.valueOf(42);
Double decimal = Double.valueOf(3.14);
Boolean flag = Boolean.TRUE;

// Constantes utiles
int max = Integer.MAX_VALUE;
int min = Integer.MIN_VALUE;
double nan = Double.NaN;
```

Las clases wrapper proporcionan metodos estaticos utiles para conversion y analisis:

```java
// Conversion de String a primitivo
int valor = Integer.parseInt("123");
double d = Double.parseDouble("3.14");
boolean b = Boolean.parseBoolean("true");

// Conversion de primitivo a String
String texto = String.valueOf(42);       // "42"
String binario = Integer.toBinaryString(10); // "1010"
String hex = Integer.toHexString(255);   // "ff"
```

## Autoboxing y unboxing

Desde Java 5, el compilador realiza conversiones automaticas entre primitivos y sus wrappers. El autoboxing convierte un primitivo a su wrapper correspondiente. El unboxing convierte un wrapper al primitivo que contiene.

```java
// Autoboxing: int -> Integer
Integer entero = 42;  // internamente Integer.valueOf(42)

// Unboxing: Integer -> int
int primitive = entero;  // internamente entero.intValue()

// En colecciones
List<Integer> numeros = new ArrayList<>();
numeros.add(10);    // autoboxing: int -> Integer
int valor = numeros.get(0);  // unboxing: Integer -> int

// Operaciones mixtas
Integer a = 5;
Integer b = 3;
int suma = a + b;  // unboxing de ambos, suma como int
```

> El autoboxing utiliza el cache de valores de `Integer.valueOf()` para enteros entre -128 y 127, reutilizando objetos ya existentes. Esto significa que `Integer a = 127; Integer b = 127; a == b` es verdadero, pero `Integer a = 128; Integer b = 128; a == b` es falso. Para comparar wrappers siempre usar `.equals()`.

## Tipos de referencia

Los tipos de referencia son todos los tipos que no son primitivos. Incluyen clases, interfaces, arrays y el tipo especial `null`. Estos valores se almacenan en el heap y las variables contienen referencias (punteros) a los objetos, no los objetos en si.

```java
String texto = "Hola";              // referencia a un String en el heap
int[] numeros = {1, 2, 3};          // referencia a un array en el heap
List<String> lista = new ArrayList<>();  // referencia a un ArrayList
Object obj = new Object();          // referencia a un Object generico
```

Cuando se asigna un tipo de referencia a otra variable, se copia la referencia, no el objeto. Ambas variables apuntan al mismo objeto en el heap:

```java
String a = "Hola";
String b = a;  // b apunta al mismo objeto que a
// Modificar el objeto a traves de 'b' afectaria a 'a' (si el objeto fuera mutable)
```

El valor por defecto de cualquier tipo de referencia no inicializado es `null`, que representa la ausencia de referencia a cualquier objeto.
