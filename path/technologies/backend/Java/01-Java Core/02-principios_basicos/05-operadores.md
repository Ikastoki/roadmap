# Operadores

Los operadores en Java son simbolos que realizan operaciones sobre uno, dos o tres operandos y producen un resultado. Java hereda la mayoria de los operadores de C/C++ y anade algunos propios como `instanceof`. Se clasifican por su aridad y funcionalidad, y su comportamiento depende de los tipos de los operandos involucrados.

## Operadores aritmeticos

Los operadores aritmeticos basicos trabajan con tipos numericos (`int`, `long`, `float`, `double`, `byte`, `short`, `char`).

| Operador | Significado      | Ejemplo      |
|----------|------------------|--------------|
| `+`      | Suma             | `a + b`      |
| `-`      | Resta            | `a - b`      |
| `*`      | Multiplicacion   | `a * b`      |
| `/`      | Division         | `a / b`      |
| `%`      | Modulo (resto)   | `a % b`      |

La division entre enteros trunca el resultado decimal:

```java
int a = 7 / 2;  // 3, no 3.5
double b = 7.0 / 2; // 3.5
double c = (double) 7 / 2; // 3.5
```

El operador `%` devuelve el resto de la division entera. Funciona con enteros y tambien con punto flotante:

```java
int resto = 10 % 3; // 1
double restoFloat = 10.5 % 3; // 1.5
```

> En Java, el signo del resultado de `%` toma el signo del dividendo: `-7 % 3` produce `-1`, no `2`.

## Operadores de asignacion

El operador basico `=` asigna el valor de la derecha a la variable de la izquierda. Los operadores compuestos combinan una operacion aritmetica con la asignacion:

```java
int x = 10;
x += 5;   // x = x + 5  -> 15
x -= 3;   // x = x - 3  -> 12
x *= 2;   // x = x * 2  -> 24
x /= 4;   // x = x / 4  -> 6
x %= 3;   // x = x % 3  -> 0
```

Estos operadores compuestos incluyen un casting implicito al tipo de la variable destino, lo que puede evitar errores de compilacion:

```java
byte b = 10;
b += 5;    // equivalente a b = (byte)(b + 5), compila
// b = b + 5; // NO compila: requiere casting explicito
```

## Operadores de comparacion

Comparan dos operandos y devuelven un valor `boolean`.

| Operador | Significado             |
|----------|-------------------------|
| `==`     | Igual a                 |
| `!=`     | Distinto de             |
| `>`      | Mayor que               |
| `<`      | Menor que               |
| `>=`     | Mayor o igual que       |
| `<=`     | Menor o igual que       |

```java
int edad = 18;
boolean esMayor = edad >= 18;  // true
boolean esMenor = edad < 18;   // false
```

Con objetos, `==` compara referencias, no contenido. Para comparar igualdad logica se usa `equals()`.

## Operadores logicos

Operan sobre valores `boolean` y soportan cortocircuito (short-circuit evaluation).

| Operador | Significado | Cortocircuito |
|----------|-------------|---------------|
| `&&`     | AND         | Si el primero es `false`, no evalua el segundo |
| `||`     | OR          | Si el primero es `true`, no evalua el segundo |
| `!`      | NOT         | No aplica    |

```java
boolean a = true;
boolean b = false;

System.out.println(a && b); // false
System.out.println(a || b); // true
System.out.println(!a);     // false
```

El cortocircuito es esencial para validaciones seguras:

```java
String s = null;
if (s != null && s.length() > 0) { // seguro: no evalua s.length() si s es null
    System.out.println("OK");
}
```

## Operadores bitwise

Trabajan a nivel de bits individuales con tipos enteros (`int`, `long`, `byte`, `short`, `char`).

| Operador | Significado   | Ejemplo                  |
|----------|---------------|--------------------------|
| `&`      | AND           | `0b1100 & 0b1010 = 0b1000` |
| `|`      | OR            | `0b1100 | 0b1010 = 0b1110` |
| `^`      | XOR           | `0b1100 ^ 0b1010 = 0b0110` |
| `~`      | Complemento   | `~0b1100 = ...11110011`  |
| `<<`     | Desplaz. izq. | `0b0011 << 2 = 0b1100`   |
| `>>`     | Desplaz. der. | `0b1100 >> 2 = 0b0011`   |
| `>>>`    | Desplaz. der. sin signo | `0b1100 >>> 2 = 0b0011` |

Los operadores `&`, `|`, `^` tambien pueden actuar como logicos no-cortocircuito sobre `boolean`, pero su uso principal es a nivel de bits:

```java
int flags = 0b1010;
int mask  = 0b0010;
boolean activo = (flags & mask) != 0; // true, el bit 2 esta activo
```

## Incremento y decremento

Los operadores `++` y `--` incrementan o decrementan la variable en 1. Existen en forma prefija y postfija, con diferente comportamiento en expresiones.

| Forma       | Nombre        | Comportamiento                        |
|-------------|---------------|---------------------------------------|
| `++x`       | Pre-incremento| Incrementa, luego usa el valor        |
| `x++`       | Post-incremento| Usa el valor, luego incrementa      |
| `--x`       | Pre-decremento | Decrementa, luego usa el valor       |
| `x--`       | Post-decremento | Usa el valor, luego decrementa      |

```java
int i = 5;
System.out.println(++i); // 6 (pre: incrementa primero)
System.out.println(i);   // 6

int j = 5;
System.out.println(j++); // 5 (post: imprime primero)
System.out.println(j);   // 6
```

## Operador ternario

El operador `? :` es el unico operador ternario de Java. Evalua una condicion y devuelve uno de dos valores:

```java
int edad = 20;
String mensaje = (edad >= 18) ? "Mayor de edad" : "Menor de edad";
```

Puede anidarse, aunque la legibilidad se resiente:

```java
String tipo = (nota >= 9) ? "Sobresaliente"
             : (nota >= 7) ? "Notable"
             : (nota >= 5) ? "Aprobado"
             : "Suspenso";
```

## instanceof

El operador `instanceof` comprueba si un objeto es de un tipo determinado (clase, subclase o interfaz):

```java
String texto = "Hola";
boolean esString = texto instanceof String;        // true
boolean esObject = texto instanceof Object;        // true
boolean esInteger = texto instanceof Integer;      // false

Object obj = "Prueba";
if (obj instanceof String s) {
    System.out.println(s.toUpperCase()); // pattern matching desde Java 16
}
```

Desde Java 16, `instanceof` admite pattern matching: si la comprobacion es exitosa, la variable se declara y castea automaticamente en el ambito del `if`.

## Precedencia de operadores

La precedencia determina el orden en que se evaluan los operadores en una expresion sin parentesis. La tabla siguiente ordena de mayor a menor prioridad:

| Nivel | Operadores                            | Asociatividad |
|-------|---------------------------------------|---------------|
| 1     | `()` `[]` `.`                         | izquierda     |
| 2     | `++` `--` `+`(unario) `-`(unario) `~` `!` | derecha    |
| 3     | `new` `(type)`                        | derecha       |
| 4     | `*` `/` `%`                           | izquierda     |
| 5     | `+` `-`                               | izquierda     |
| 6     | `<<` `>>` `>>>`                       | izquierda     |
| 7     | `<` `<=` `>` `>=` `instanceof`        | izquierda     |
| 8     | `==` `!=`                             | izquierda     |
| 9     | `&`                                   | izquierda     |
| 10    | `^`                                   | izquierda     |
| 11    | `|`                                   | izquierda     |
| 12    | `&&`                                  | izquierda     |
| 13    | `||`                                  | izquierda     |
| 14    | `? :`                                 | derecha       |
| 15    | `=` `+=` `-=` `*=` `/=` `%=` etc.     | derecha       |

```java
int resultado = 5 + 3 * 2; // 11, no 16 (* tiene mayor precedencia)
boolean logico = 5 > 3 && 2 < 4; // true (relacionales antes que logicos)
```

Ante la duda, el uso de parentesis es siempre preferible a confiar en la precedencia. El codigo se lee mas veces de las que se escribe, y los parentesis explicitos eliminan ambiguedades para el lector humano.
