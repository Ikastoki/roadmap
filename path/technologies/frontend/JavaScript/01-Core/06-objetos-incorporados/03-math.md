<h1 align='center'>Math</h1>

<h2>📑 Contenido</h2>

- [Math](#math)
- [Propiedades Estáticas de Math](#propiedades-estáticas-de-math)
  - [Math.E](#mathe)
  - [Math.LN2](#mathln2)
  - [Math.LN10](#mathln10)
  - [Math.LOG2E](#mathlog2e)
  - [Math.LOG10E](#mathlog10e)
  - [Math.PI](#mathpi)
  - [Math.SQRT1\_2](#mathsqrt1_2)
  - [Math.SQRT2](#mathsqrt2)
- [Métodos de Redondeo](#métodos-de-redondeo)
  - [Math.ceil()](#mathceil)
  - [Math.floor()](#mathfloor)
  - [Math.round()](#mathround)
  - [Math.trunc()](#mathtrunc)
  - [Math.random()](#mathrandom)
- [Métodos Aritméticos](#métodos-aritméticos)
  - [Math.abs()](#mathabs)
  - [Math.sign()](#mathsign)
  - [Math.max()](#mathmax)
  - [Math.min()](#mathmin)
- [Métodos Exponenciales y Logarítmicos](#métodos-exponenciales-y-logarítmicos)
  - [Math.exp()](#mathexp)
  - [Math.expm1()](#mathexpm1)
  - [Math.log()](#mathlog)
  - [Math.log1p()](#mathlog1p)
  - [Math.log2()](#mathlog2)
  - [Math.pow()](#mathpow)
  - [Math.sqrt()](#mathsqrt)
- [Métodos Trigonométricos](#métodos-trigonométricos)
  - [Math.acos()](#mathacos)
  - [Math.acosh()](#mathacosh)
  - [Math.asin()](#mathasin)
  - [Math.asinh()](#mathasinh)
  - [Math.atan()](#mathatan)
  - [Math.atan2()](#mathatan2)
  - [Math.atanh()](#mathatanh)
  - [Math.cos()](#mathcos)
  - [Math.cosh()](#mathcosh)
  - [Math.sin()](#mathsin)
  - [Math.sinh()](#mathsinh)
  - [Math.tan()](#mathtan)
  - [Math.tanh()](#mathtanh)
- [Métodos Misceláneos](#métodos-misceláneos)
  - [Math.cbrt()](#mathcbrt)
  - [Math.clz32()](#mathclz32)
  - [Math.fround()](#mathfround)
  - [Math.hypot()](#mathhypot)
  - [Math.imul()](#mathimul)

## Math

El objeto `Math` en JavaScript proporciona una colección de propiedades y métodos que permiten realizar operaciones matemáticas avanzadas y comunes de manera sencilla. Es una herramienta esencial para trabajar con cálculos aritméticos, exponenciales, trigonométricos y otros tipos de funciones matemáticas. Este capítulo detalla las propiedades estáticas y los métodos del objeto `Math` divididos en categorías, lo que proporciona una base sólida para comprender cómo realizar diferentes operaciones matemáticas en JavaScript.

## Propiedades Estáticas de Math

El objeto `Math` contiene varias propiedades estáticas que representan importantes constantes matemáticas. Estas propiedades no pueden ser modificadas y se utilizan frecuentemente en cálculos matemáticos.

### Math.E

La constante de Euler, base de los logaritmos naturales, aproximadamente 2.718.

```js
console.log(Math.E); // Imprime: 2.718281828459045
```

### Math.LN2

El logaritmo natural de 2, aproximadamente 0.693.

```js
console.log(Math.LN2); // Imprime: 0.6931471805599453
```

### Math.LN10

El logaritmo natural de 10, aproximadamente 2.302.

```js
console.log(Math.LN10); // Imprime: 2.302585092994046
```

### Math.LOG2E

El logaritmo en base 2 de Math.E, aproximadamente 1.442.

```js
console.log(Math.LOG2E); // Imprime: 1.4426950408889634
``;
```

### Math.LOG10E

El logaritmo en base 10 de Math.E, aproximadamente 0.434.

```js
console.log(Math.LOG10E); // Imprime: 0.4342944819032518
```

### Math.PI

El valor de pi (π), la relación entre la circunferencia y el diámetro de un círculo, aproximadamente 3.14159.

```js
console.log(Math.PI); // Imprime: 3.141592653589793
```

### Math.SQRT1_2

La raíz cuadrada de 1/2, aproximadamente 0.707.

```js
console.log(Math.SQRT1_2); // Imprime: 0.7071067811865476
```

### Math.SQRT2

La raíz cuadrada de 2, aproximadamente 1.414.

```js
console.log(Math.SQRT2); // Imprime: 1.4142135623730951
```

## Métodos de Redondeo

Los métodos de redondeo en `Math` son esenciales para manipular valores decimales, redondeándolos hacia arriba, hacia abajo o según el número más cercano.

### Math.ceil()

Redondea un número hacia arriba al entero más cercano.

```js
console.log(Math.ceil(4.2)); // Imprime: 5
```

### Math.floor()

Redondea un número hacia abajo al entero más cercano.

```js
console.log(Math.floor(4.8)); // Imprime: 4
```

### Math.round()

Redondea un número al entero más cercano. Si el número está a medio camino entre dos enteros, redondea hacia arriba.

```js
console.log(Math.round(4.5)); // Imprime: 5
console.log(Math.round(4.4)); // Imprime: 4
```

### Math.trunc()

Elimina la parte decimal de un número y devuelve solo la parte entera.

```js
console.log(Math.trunc(4.9)); // Imprime: 4
```

### Math.random()

Devuelve un número pseudoaleatorio entre 0 (inclusive) y 1 (exclusivo).

```js
console.log(Math.random()); // Imprime: un número entre 0 y 1
```

> [!NOTE]
> El Math.random no es para redondear pero se utiliza mucho junto a estos métodos.

## Métodos Aritméticos

Estos métodos aritméticos permiten realizar operaciones comunes como obtener valores absolutos, determinar el signo de un número o encontrar el máximo o mínimo entre varios números.

### Math.abs()

Devuelve el valor absoluto de un número (es decir, su valor sin signo).

```js
console.log(Math.abs(-10)); // Imprime: 10
```

### Math.sign()

Devuelve el signo de un número: `1` si es positivo, `-1` si es negativo, y `0` si es exactamente cero.

```js
console.log(Math.sign(-3)); // Imprime: -1
console.log(Math.sign(0)); // Imprime: 0
console.log(Math.sign(5)); // Imprime: 1
```

### Math.max()

Devuelve el valor máximo de una lista de números.

```js
console.log(Math.max(1, 2, 3, 4, 5)); // Imprime: 5
```

### Math.min()

Devuelve el valor mínimo de una lista de números.

```js
console.log(Math.min(1, 2, 3, 4, 5)); // Imprime: 1
```

## Métodos Exponenciales y Logarítmicos

Estos métodos permiten trabajar con exponentes y logaritmos, herramientas fundamentales en el análisis matemático y la programación.

### Math.exp()

Devuelve el valor de `e` elevado a la potencia de un número dado.

```js
console.log(Math.exp(1)); // Imprime: 2.718281828459045 (valor de Math.E)
```

### Math.expm1()

Devuelve el valor de `e^x - 1`, donde `x` es el argumento dado.

```js
console.log(Math.expm1(1)); // Imprime: 1.718281828459045
```

### Math.log()

Devuelve el logaritmo natural de un número (base `e`).

```js
console.log(Math.log(1)); // Imprime: 0
```

### Math.log1p()

Devuelve el logaritmo natural de `1 + x`.

```js
console.log(Math.log1p(1)); // Imprime: 0.6931471805599453
```

### Math.log2()

Devuelve el logaritmo en base 2 de un número.

```js
console.log(Math.log2(8)); // Imprime: 3
```

### Math.pow()

Devuelve la base elevada a la potencia del exponente especificado.

```js
console.log(Math.pow(2, 3)); // Imprime: 8 (2^3)
```

### Math.sqrt()

Devuelve la raíz cuadrada de un número.

```js
console.log(Math.sqrt(16)); // Imprime: 4
```

## Métodos Trigonométricos

Los métodos trigonométricos de Math permiten realizar cálculos relacionados con ángulos y triángulos. Trabajan principalmente en radianes.

### Math.acos()

Devuelve el arcocoseno (inverso del coseno) de un número.

```js
console.log(Math.acos(1)); // Imprime: 0
```

### Math.acosh()

Devuelve el coseno hiperbólico inverso de un número.

```js
console.log(Math.acosh(1)); // Imprime: 0
```

### Math.asin()

Devuelve el arcoseno (inverso del seno) de un número.

```js
console.log(Math.asin(0)); // Imprime: 0
```

### Math.asinh()

Devuelve el seno hiperbólico inverso de un número.

```js
console.log(Math.asinh(1)); // Imprime: 0.881373587019543
```

### Math.atan()

Devuelve el arcotangente (inverso de la tangente) de un número.

```js
console.log(Math.atan(1)); // Imprime: 0.7853981633974483 (π/4)
```

### Math.atan2()

Devuelve el arcotangente de las coordenadas `y` y`x`, en radianes. Es útil para convertir coordenadas cartesianas a coordenadas polares.

```js
console.log(Math.atan2(1, 1)); // Imprime: 0.7853981633974483 (π/4)
```

### Math.atanh()

Devuelve la tangente hiperbólica inversa de un número.

```js
console.log(Math.atanh(0.5)); // Imprime: 0.5493061443340549
```

### Math.cos()

Devuelve el coseno de un número (en radianes).

```js
console.log(Math.cos(Math.PI / 3)); // Imprime: 0.5
```

### Math.cosh()

Devuelve el coseno hiperbólico de un número.

```js
console.log(Math.cosh(0)); // Imprime: 1
```

### Math.sin()

Devuelve el seno de un número (en radianes).

```js
console.log(Math.sin(Math.PI / 2)); // Imprime: 1
```

### Math.sinh()

Devuelve el seno hiperbólico de un número.

```js
console.log(Math.sinh(0)); // Imprime: 0
```

### Math.tan()

Devuelve la tangente de un número (en radianes).

```js
console.log(Math.tan(Math.PI / 4)); // Imprime: 1
```

### Math.tanh()

Devuelve la tangente hiperbólica de un número.

```js
console.log(Math.tanh(0)); // Imprime: 0
```

## Métodos Misceláneos

Estos métodos diversos ofrecen soluciones útiles para operaciones menos comunes.

### Math.cbrt()

Devuelve la raíz cúbica de un número.

```js
console.log(Math.cbrt(27)); // Imprime: 3
```

### Math.clz32()

Devuelve el número de ceros a la izquierda en la representación binaria de un número entero de 32 bits.

```js
console.log(Math.clz32(1)); // Imprime: 31
```

### Math.fround()

Devuelve el valor flotante simple (32 bits) más cercano de un número.

```js
console.log(Math.fround(5.5)); // Imprime: 5.5
```

### Math.hypot()

Devuelve la raíz cuadrada de la suma de los cuadrados de sus argumentos (útil para calcular la longitud de la hipotenusa en un triángulo rectángulo).

```js
console.log(Math.hypot(3, 4)); // Imprime: 5
```

### Math.imul()

Realiza la multiplicación de dos enteros de 32 bits y devuelve el resultado, considerando desbordamiento en los bits de 32 bits.

```js
console.log(Math.imul(2, 4)); // Imprime: 8
```
