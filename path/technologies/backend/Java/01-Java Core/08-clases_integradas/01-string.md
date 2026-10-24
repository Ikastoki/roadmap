# String

La clase `String` en Java representa cadenas de caracteres inmutables. Toda cadena literal escrita entre comillas dobles en el codigo fuente se convierte en una instancia de `String`. La inmutablebilidad no es un accidente de diseno, sino una decision deliberada que impacta en la seguridad, el rendimiento y la arquitectura de concurrencia de la plataforma.

## Inmutablebilidad

Una vez creado, un objeto `String` no puede modificarse. Cualquier operacion que parezca alterar la cadena -- como `toUpperCase()` o `replace()` -- devuelve un nuevo objeto `String` sin modificar el original. Esto permite que las instancias de `String` sean compartidas de forma segura entre hilos sin necesidad de sincronizacion, y que se usen como claves en mapas hash sin riesgo de que su valor hash cambie.

```java
String s = "hola";
String t = s.toUpperCase(); // s sigue siendo "hola", t es "HOLA"
```

> La inmutablebilidad tambien permite el String pool, una optimizacion que evita crear multiples objetos para cadenas identicas.

## String pool y memoria heap

El String pool (tambien llamado intern pool) es un area especial dentro del heap de la JVM donde se almacenan literales de cadena. Cuando el compilador encuentra un literal, la JVM verifica si ya existe en el pool; si existe, reutiliza la referencia. Si no, crea una nueva entrada.

```java
String a = "java";
String b = "java";
System.out.println(a == b); // true, mismo objeto en el pool

String c = new String("java");
System.out.println(a == c); // false, c esta en heap fuera del pool
System.out.println(a.equals(c)); // true, mismo contenido
```

El metodo `intern()` permite insertar manualmente una cadena en el pool:

```java
String d = c.intern();
System.out.println(a == d); // true
```

## Metodos principales

### Longitud y acceso

```java
String s = "Hello World";
int len = s.length();               // 11
char ch = s.charAt(0);              // 'H'
String sub = s.substring(0, 5);     // "Hello"
String sub2 = s.substring(6);       // "World"
```

### Busqueda

```java
int idx1 = s.indexOf('o');          // 4
int idx2 = s.indexOf("World");      // 6
int idx3 = s.lastIndexOf('o');      // 7
boolean b = s.contains("Hello");    // true
boolean starts = s.startsWith("He"); // true
boolean ends = s.endsWith("ld");    // true
```

### Comparacion

```java
s.equals("Hello World");             // true
s.equalsIgnoreCase("hello world");   // true
int cmp = s.compareTo("Hello World");// 0
int cmp2 = s.compareToIgnoreCase("hello world"); // 0
```

### Transformacion

```java
String mayus = s.toUpperCase();      // "HELLO WORLD"
String minus = s.toLowerCase();      // "hello world"
String trimmed = "  hola  ".trim();  // "hola"
String reemplazado = s.replace('l', 'x'); // "Hexxo Worxd"
String reemplazado2 = s.replace("World", "Java"); // "Hello Java"
String[] partes = s.split(" ");      // ["Hello", "World"]
String unido = String.join("-", "a", "b", "c"); // "a-b-c"
```

### Conversion

```java
String num = String.valueOf(42);     // "42"
String bool = String.valueOf(true);  // "true"
String fromChars = String.valueOf(new char[]{'a','b'}); // "ab"
char[] chars = s.toCharArray();
```

## StringBuilder y StringBuffer

Dado que `String` es inmutable, la concatenacion repetitiva con `+` genera muchos objetos temporales. Para construccion eficiente de cadenas existen `StringBuilder` (no sincronizado, mas rapido) y `StringBuffer` (sincronizado, thread-safe).

```java
StringBuilder sb = new StringBuilder();
sb.append("Hola ");
sb.append("Mundo ");
sb.append(42);
String resultado = sb.toString(); // "Hola Mundo 42"
```

Metodos principales:

```java
sb.insert(5, "cruel ");
sb.delete(0, 5);
sb.reverse();
sb.replace(0, 2, "xx");
sb.length();
sb.setLength(0); // vaciar
```

`StringBuffer` tiene la misma API pero todos sus metodos son `synchronized`, lo que introduce sobrecarga. En contextos monohilo siempre debe preferirse `StringBuilder`.

> Desde Java 5, la concatenacion con `+` se compila internamente usando `StringBuilder` en el bytecode, pero en bucles sigue siendo ineficiente porque se crea un nuevo `StringBuilder` en cada iteracion.

## String vs char[]

Antes de Java, las cadenas se manejaban como arrays de caracteres terminados en nulo. `String` ofrece ventajas decisivas:

- **Inmutablebilidad**: no se puede modificar accidentalmente el contenido.
- **Seguridad**: ideal para contrasenas -- aunque por seguridad deberia usarse `char[]` para contrasenas porque `String` permanece en el pool hasta que el GC lo recolecte.
- **API rica**: metodos integrados para busqueda, comparacion y transformacion.
- **Sobrecarga**: cada `String` es un objeto con overhead; `char[]` es mas ligero.

Para contrasenas, la recomendacion de seguridad es usar `char[]` y limpiarlo explicitamente:

```java
char[] password = obtenerPassword();
// usar password
Arrays.fill(password, '0'); // limpiar
```
