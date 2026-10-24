# Matchers basicos

Los matchers basicos de Hamcrest cubren las operaciones fundamentales de igualdad, identidad, nulidad y comparacion. Se definen como metodos estaticos en la clase `org.hamcrest.Matchers` y se importan mediante `import static org.hamcrest.Matchers.*`.

## is

`is` es un matcher decorador que no anade logica propia, sino que envuelve a otro matcher para mejorar la legibilidad. Se lee como "es igual a" o "es". Tambien tiene una sobrecarga que recibe un valor directamente y lo convierte en `is(equalTo(valor))`.

```java
assertThat("texto", is("texto"));
assertThat(42, is(equalTo(42)));
```

Aunque funcionalmente redundante, `is` hace que la asercion se lea como una oracion: "assert that X is Y".

## equalTo

`equalTo` verifica la igualdad logica mediante `Object.equals()`. Es el matcher mas utilizado y el que esta implicito cuando se pasa un valor directamente a `assertThat`.

```java
assertThat("Hola", equalTo("Hola"));
assertThat(100, equalTo(100));
assertThat(3.14, equalTo(3.14));
```

> `equalTo` usa el metodo `equals` del objeto. Si la clase bajo prueba no sobrescribe `equals`, la comparacion se hace por identidad de referencia, lo que puede producir falsos positivos o negativos inesperados.

## not

`not` niega el matcher que recibe como argumento. Es el equivalente logico del operador `!` pero aplicado a predicados.

```java
assertThat("Hola", not("Adios"));
assertThat("Hola", not(equalTo("Adios")));
assertThat(Arrays.asList("a"), not(empty()));
```

Puede combinarse con cualquier otro matcher: `not(containsString("x"))`, `not(greaterThan(10))`, etc.

## nullValue / notNullValue

`nullValue` verifica que el valor sea `null`. `notNullValue` verifica que no lo sea. Son semanticamente equivalentes a `assertNull` y `assertNotNull` de JUnit.

```java
Object nulo = null;
Object noNulo = new Object();

assertThat(nulo, nullValue());
assertThat(noNulo, notNullValue());
assertThat(noNulo, is(notNullValue()));
```

Ambos matchers aceptan un parametro opcional de tipo para asistir al compilador en la inferencia de tipos en casos genericos:

```java
assertThat(optional.orElse(null), nullValue(String.class));
```

## sameInstance

`sameInstance` verifica que dos referencias apunten al mismo objeto en memoria (operador `==`). Su negacion es `not(sameInstance(...))` o directamente `theInstance`.

```java
String a = "cadena";
String b = a;
String c = new String("cadena");

assertThat(b, sameInstance(a));  // pasa: misma referencia
assertThat(c, not(sameInstance(a))); // pasa: distinta referencia
```

## greaterThan / lessThan / greaterThanOrEqualTo / lessThanOrEqualTo

Matchers para comparacion numerica. Funcionan con cualquier tipo que implemente `Comparable<T>`.

```java
assertThat(10, greaterThan(5));
assertThat(5, lessThan(10));
assertThat(10, greaterThanOrEqualTo(10));
assertThat(5, lessThanOrEqualTo(5));
```

Para valores `double` y `float`, se debe tener precaucion con la precision de punto flotante. En esos casos es preferible usar `closeTo`.

## closeTo

`closeTo` verifica que un valor numerico (double o float) se encuentre dentro de un margen de tolerancia respecto a un valor esperado. Es la alternativa segura a `equalTo` para numeros de punto flotante.

```java
assertThat(3.14159, closeTo(3.14, 0.01));
assertThat(100.0, closeTo(100.0, 0.001));
```

El primer argumento es el valor esperado, el segundo es el margen de error maximo permitido (delta). La asercion pasa si `Math.abs(valorEsperado - valorReal) <= delta`.

## hasToString

`hasToString` verifica el resultado del metodo `toString()` del objeto. Acepta tanto un `String` literal como un matcher para validaciones parciales.

```java
Object obj = new Object() {
    @Override
    public String toString() {
        return "MiObjeto{id=42}";
    }
};

assertThat(obj, hasToString("MiObjeto{id=42}"));
assertThat(obj, hasToString(containsString("id=42")));
```

```java
// Resumen de matchers basicos
assertThat(valor, is(equalTo(esperado)));
assertThat(valor, is(not(esperado)));
assertThat(valor, nullValue());
assertThat(valor, notNullValue());
assertThat(valor, sameInstance(otro));
assertThat(valor, greaterThan(limite));
assertThat(valor, lessThan(limite));
assertThat(valor, closeTo(esperado, delta));
assertThat(valor, hasToString(contiene));
```
