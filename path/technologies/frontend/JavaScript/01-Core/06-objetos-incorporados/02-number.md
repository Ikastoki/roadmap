<h1 align='center'>Number</h1>

<h2>📑 Contenido</h2>

- [Number](#number)
- [Propiedades Estáticas de Number](#propiedades-estáticas-de-number)
  - [Number.EPSILON](#numberepsilon)
  - [Number.MAX\_SAFE\_INTEGER](#numbermax_safe_integer)
  - [Number.MAX\_VALUE](#numbermax_value)
  - [Number.MIN\_SAFE\_INTEGER](#numbermin_safe_integer)
  - [Number.MIN\_VALUE](#numbermin_value)
  - [Number.NaN](#numbernan)
  - [Number.NEGATIVE\_INFINITY](#numbernegative_infinity)
  - [Number.POSITIVE\_INFINITY](#numberpositive_infinity)
- [Métodos Estáticos de Number](#métodos-estáticos-de-number)
  - [Number.isFinite()](#numberisfinite)
  - [Number.isInteger()](#numberisinteger)
  - [Number.isNaN()](#numberisnan)
  - [Number.isSafeInteger()](#numberissafeinteger)
  - [Number.parseFloat()](#numberparsefloat)
  - [Number.parseInt()](#numberparseint)
- [Métodos de Instancia de Number](#métodos-de-instancia-de-number)
  - [toExponential()](#toexponential)
  - [toFixed()](#tofixed)
  - [toLocaleString()](#tolocalestring)
  - [toPrecision()](#toprecision)
  - [toString()](#tostring)
  - [valueOf()](#valueof)
- [Local String](#local-string)
  - [Number.prototype.toLocaleString()](#numberprototypetolocalestring)

## Number

El objeto `Number` en JavaScript representa valores numéricos y proporciona varias propiedades y métodos útiles tanto a nivel estático como de instancia. Es una herramienta poderosa para realizar operaciones matemáticas precisas y para manipular números en distintas formas de representación. Este capítulo explora tanto las propiedades y métodos estáticos como los métodos de instancia del objeto `Number`, fundamentales para el trabajo numérico en JavaScript.

## Propiedades Estáticas de Number

Las propiedades estáticas de `Number` están asociadas directamente con el objeto `Number` y no con instancias específicas de números. Estas propiedades son útiles para entender los límites numéricos de JavaScript y para manejar errores relacionados con cálculos de precisión.

### Number.EPSILON

La menor diferencia positiva entre dos representaciones de punto flotante distintas en JavaScript. Es útil para comparar números con decimales y evitar problemas de precisión.

```js
console.log(Number.EPSILON); // Imprime: 2.220446049250313e-16
```

### Number.MAX_SAFE_INTEGER

El valor máximo seguro que se puede representar como un entero en JavaScript (2^53 - 1). Cualquier entero mayor que este podría causar imprecisiones.

```js
console.log(Number.MAX_SAFE_INTEGER); // Imprime: 9007199254740991
```

### Number.MAX_VALUE

El valor numérico más grande que puede representarse en JavaScript. Este es un número muy grande, cercano a 1.79e+308.

```js
console.log(Number.MAX_VALUE); // Imprime: 1.7976931348623157e+308
```

### Number.MIN_SAFE_INTEGER

El valor más pequeño seguro que se puede representar como un entero en JavaScript (-2^53 + 1).

```js
console.log(Number.MIN_SAFE_INTEGER); // Imprime: -9007199254740991
```

### Number.MIN_VALUE

El valor positivo más pequeño mayor que cero que se puede representar en JavaScript, aproximadamente 5e-324.

```js
console.log(Number.MIN_VALUE); // Imprime: 5e-324
```

### Number.NaN

Representa el valor especial `NaN` (Not-a-Number). Se produce principalmente en operaciones matemáticas no válidas como dividir cero entre cero.

```js
console.log(Number.NaN); // Imprime: NaN
```

### Number.NEGATIVE_INFINITY

Representa el valor negativo del infinito. Se obtiene cuando un número es menor que el límite inferior de lo que JavaScript puede representar.

```js
console.log(Number.NEGATIVE_INFINITY); // Imprime: -Infinity
```

### Number.POSITIVE_INFINITY

Representa el valor positivo del infinito. Se produce cuando un número excede el valor máximo representable.

```js
console.log(Number.POSITIVE_INFINITY); // Imprime: Infinity
```

## Métodos Estáticos de Number

Los métodos estáticos de `Number` están asociados con el objeto `Number` y son útiles para realizar verificaciones y conversiones.

### Number.isFinite()

Verifica si un valor es un número finito. Devuelve `true` si el número es finito y `false` si es `Infinity`, `-Infinity` o `NaN`.

```js
console.log(Number.isFinite(100)); // Imprime: true
console.log(Number.isFinite(Infinity)); // Imprime: false
```

### Number.isInteger()

Determina si el valor es un número entero. Devuelve `true` para números enteros y `false` para números de punto flotante o no numéricos.

```js
console.log(Number.isInteger(10)); // Imprime: true
console.log(Number.isInteger(10.5)); // Imprime: false
```

### Number.isNaN()

Verifica si el valor es `NaN` (Not-a-Number). A diferencia de `isNaN()`, este método no convierte el valor a número antes de la verificación.

```js
console.log(Number.isNaN(NaN)); // Imprime: true
console.log(Number.isNaN(10)); // Imprime: false
```

### Number.isSafeInteger()

Verifica si el valor es un número entero dentro de los límites de los valores seguros (`Number.MIN_SAFE_INTEGER` a` Number.MAX_SAFE_INTEGER`).

```js
console.log(Number.isSafeInteger(9007199254740991)); // Imprime: true
console.log(Number.isSafeInteger(9007199254740992)); // Imprime: false
```

### Number.parseFloat()

Convierte una cadena de texto en un número de punto flotante. Si no puede convertir el valor, devuelve `NaN`.

```js
console.log(Number.parseFloat("3.14")); // Imprime: 3.14
```

### Number.parseInt()

Convierte una cadena de texto en un número entero, eliminando cualquier parte fraccionaria y caracteres no numéricos. También se puede especificar una base (radix).

```js
console.log(Number.parseInt("42")); // Imprime: 42
console.log(Number.parseInt("42px")); // Imprime: 42
console.log(Number.parseInt("101", 2)); // Imprime: 5 (base 2)
```

## Métodos de Instancia de Number

Los métodos de instancia de `Number` se aplican a números específicos. Estos métodos permiten formatear y representar los números de varias formas.

### toExponential()

Devuelve una cadena que representa el número en notación exponencial, con un número de decimales especificado.

```js
const numero = 123456;
console.log(numero.toExponential(2)); // Imprime: "1.23e+5"
```

### toFixed()

Devuelve una cadena representando el número con un número fijo de decimales. El resultado está redondeado si es necesario.

```js
const pi = 3.141592;
console.log(pi.toFixed(2)); // Imprime: "3.14"
```

### toLocaleString()

Devuelve una cadena con el número formateado de acuerdo con la configuración regional específica. Puede personalizarse con opciones como el estilo de moneda.

```js
const numero = 123456.789;
console.log(numero.toLocaleString("es-ES")); // Imprime: "123.456,789"
```

### toPrecision()

Devuelve una cadena que representa el número con una precisión total especificada, redondeando el número si es necesario.

```js
const numero = 123.456;
console.log(numero.toPrecision(4)); // Imprime: "123.5"
```

### toString()

Convierte el número en una cadena de texto. Puede recibir un argumento opcional que especifica la base numérica (radix) para representar el número.

```js
const numero = 255;
console.log(numero.toString(16)); // Imprime: "ff" (base 16)
```

### valueOf()

Devuelve el valor primitivo del objeto `Number`. Aunque normalmente JavaScript convierte automáticamente los números cuando es necesario, este método puede ser útil en algunas circunstancias.

## Local String

### Number.prototype.toLocaleString()

Permite convertir un número en una cadena localizable, permitiendo aplicar diferentes estilos de formato, como moneda o porcentajes.

```js
const numero = 123456.789;
console.log(
  numero.toLocaleString("de-DE", { style: "currency", currency: "EUR" })
);
// Imprime: "123.456,79 €"
```
