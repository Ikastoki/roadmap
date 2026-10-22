<h1 align="center">Operadores</h1>

<h2>📑 Contenido</h2>

- [Operadores](#operadores)
  - [Operadores Aritméticos](#operadores-aritméticos)
  - [Operadores de Comparación](#operadores-de-comparación)
  - [Operadores de Asignación](#operadores-de-asignación)
  - [Operadores Lógicos](#operadores-lógicos)
  - [Operador Fusión Nula (nullish coalescing)](#operador-fusión-nula-nullish-coalescing)
  - [Operadores de Concatenación de Cadenas](#operadores-de-concatenación-de-cadenas)
  - [Operadores de Incremento y Decremento](#operadores-de-incremento-y-decremento)
  - [Operadores Ternarios](#operadores-ternarios)
  - [Operadores Unarios](#operadores-unarios)
  - [Operadores Bitwise](#operadores-bitwise)
  - [Operador Tilde](#operador-tilde)
  - [Operador de Propagación (Spread Operator)](#operador-de-propagación-spread-operator)
  - [Operador Rest](#operador-rest)
- [Ejemplos Extra](#ejemplos-extra)

## Operadores

En JavaScript, los operadores son símbolos o palabras clave que permiten realizar operaciones con valores o variables, y manipular datos. Estos operadores están clasificados en diferentes categorías según su propósito, como los operadores aritméticos, de comparación, de asignación, lógicos, entre otros. En este capítulo, exploraremos los operadores más utilizados y algunas curiosidades que los rodean.

### Operadores Aritméticos

Los operadores aritméticos se utilizan para realizar cálculos matemáticos.

- `+` (Suma)
- `-` (Resta)
- `*` (Multiplicación)
- `/ `(División)
- `%` (Módulo, devuelve el resto de una división)
- `**` (Exponente)

```js
// Ejemplo suma
let a = 5;
let b = 3;
let suma = a + b; // suma es igual a 8
```

### Operadores de Comparación

Los operadores de comparación comparan dos valores y devuelven un valor booleano (`true` o `false`).

- `==` (Igual a)
- `!=` (No igual a)
- `===` (Igual a estricto, verifica igualdad de valor y tipo de dato)
- `!==` (No igual a estricto)
- `>` (Mayor que)
- `<` (Menor que)
- `>=` (Mayor o igual que)
- `<=` (Menor o igual que)

```js
//Comparar números
let a = 5;
let b = "5";
a == b; // true, valores iguales
a === b; // false, tipos de datos diferentes
```

> [!NOTE]
> La diferencia entre == y === radica en que el primero realiza coerción de tipo, mientras que el segundo requiere que los valores sean del mismo tipo y valor.

### Operadores de Asignación

Los operadores de asignación se usan para asignar valores a variables.

- `=` (Asignación)
- `+=, -=, \*=, /=` (Operadores de asignación compuesta)

```js
// Asignar un valor
let x = 10;
x += 5; // x ahora es igual a 15
```

### Operadores Lógicos

Los operadores lógicos permiten combinar valores booleanos.

El operador && devuelve el primer valor falso que encuentra, o el último valor si no hay falsos.

El operador || funciona de manera similar, pero en lugar de buscar un valor falso, busca el primer valor verdadero

- `&&` (Y lógico)
- `||` (O lógico)
- `!` (Negación lógica)

```js
// Comparar edades
let edad = 20;
let esMayorDeEdad = edad >= 18 && edad <= 65; // esMayorDeEdad es true

// &&: Devuelve el primer valor falsy encontrado o el último valor si todos son truthy.
console.log(0 && "Hola"); // 0
console.log("Hola" && 5); // 5

// ||: Devuelve el primer valor truthy encontrado o el último valor si todos son falsy.
console.log(0 || "Hola"); // "Hola"
console.log(false || 5); // 5
console.log(null || undefined); // undefined
```

### Operador Fusión Nula (nullish coalescing)

El operador fusión nula `(??)` es un operador lógico que devuelve el operando de la derecha si el operando de la izquierda es `null` o `undefined`. De lo contrario, devuelve el operando de la izquierda. Es útil para establecer valores predeterminados sin sobrescribir otros valores `falsy`.

```js
let valor1 = null;
let valor2 = "Hola";

let resultado = valor1 ?? valor2; // "Hola"
```

A diferencia del operador `||`, que devuelve el primer valor `truthy`, el operador `??` solo se ocupa de `null` y `undefined`, lo que lo hace más específico para ciertas situaciones.

### Operadores de Concatenación de Cadenas

El operador + se utiliza para concatenar cadenas de texto (strings).

Si uno de los operandos es una cadena, JavaScript convertirá los demás operandos a cadena antes de concatenar

- `+` (Concatenación de cadenas)

```js
//Concatenar Cadena
let nombre = "Juan";
let apellido = "Pérez";
let nombreCompleto = nombre + " " + apellido; // "Juan Pérez"
```

### Operadores de Incremento y Decremento

Se utilizan para aumentar o disminuir en una unidad el valor de una variable.

Los operadores ++ y -- tienen formas prefijas y postfijas, y el resultado puede variar según el uso:

- `++` (Incremento)
- `--` (Decremento)

```js
// Util para crear un contador.(Utilizados en while)
let contador = 0;
contador++; // Incrementa en 1, contador es igual a 1

// Prefijo y postfijo
let x = 5;
console.log(x++); // 5, incrementa después de la evaluación
console.log(++x); // 7, incrementa antes de la evaluación
```

### Operadores Ternarios

El operador ternario es una forma corta de realizar una evaluación condicional. La sintaxis es:

`condición ? valorSiVerdadero : valorSiFalso`

```js
//Condicional para determinar si es mayor o menor de edad.
let edad = 25;
let mensaje = edad >= 18 ? "Es mayor de edad" : "Es menor de edad";
```

### Operadores Unarios

Los operadores unarios son aquellos que actúan sobre un solo operando.

- - (Conversión a número)
- - (Negación numérica)
- typeof (Tipo del operando)
- delete (Elimina una propiedad de un objeto)

```js
console.log(typeof "hola"); // "string"
let objeto = { nombre: "Juan" };
delete objeto.nombre;
console.log(objeto); // {}
```

### Operadores Bitwise

Los operadores bitwise (o de manipulación de bits) permiten operar directamente sobre los bits individuales de números enteros. Estos operadores tratan a los números como si fueran representados en binario (32 bits), lo que permite realizar operaciones a nivel de bit.

| Operador | Nombre                 | Descripción                                                            |
|----------|------------------------|------------------------------------------------------------------------|
| `&`      | AND                    | Devuelve 1 si ambos bits son 1                                         |
| `|`      | OR                     | Devuelve 1 si al menos uno de los bits es 1                            |
| `^`      | XOR (OR exclusivo)     | Devuelve 1 si los bits son diferentes                                  |
| `~`      | NOT                    | Invierte todos los bits del número                                     |
| `<<`     | Shift izquierdo        | Desplaza los bits a la izquierda, multiplicando por potencias de 2     |
| `>>`     | Shift derecho          | Desplaza los bits a la derecha, manteniendo el signo                   |
| `>>>`    | Shift derecho sin signo| Desplaza los bits a la derecha, rellenando con ceros desde la izquierda|


### Operador Tilde

El operador `~` es el Bitwise NOT (negación bit a bit). Convierte su operando en un entero de 32 bits y luego invierte todos sus bits.

```javascript
console.log(~5); // -6
```

¿Por qué?

- El número 5 en binario es 00000000 00000000 00000000 00000101

- Al aplicar `~`, se invierten los bits: 11111111 11111111 11111111 11111010

- Esto representa -6 en complemento a dos

### Operador de Propagación (Spread Operator)

Permite descomponer o expandir elementos de un iterable (como un array o un objeto iterable) en lugares donde se esperan múltiples elementos. Se representa mediante tres puntos suspensivos (`...`).

```js
//Copiar y combinar un arreglo.
let numeros = [1, 2, 3];
let otrosNumeros = [...numeros, 4, 5]; // [1, 2, 3, 4, 5]

//Clonar un objeto
let originalObjeto = { nombre: "Juan", edad: 30 };
let copiaObjeto = { ...originalObjeto };
console.log(copiaObjeto); // { nombre: "Juan", edad: 30 }
```

### Operador Rest

El operador rest, también representado por tres puntos suspensivos (...), se utiliza para recoger argumentos restantes en funciones o para recoger elementos restantes en destructuración de arrays o objetos.

> [!NOTE]
> A diferencia del operador de propagación(spread), que se usa para expandir elementos, el operador rest se utiliza para agrupar elementos restantes.
>
> - **Spread:** Expande elementos.
> - **Rest:** Agrupa elementos.

```js
// Recoge todos los elementos restantes después de primero y segundo en un array llamado resto.
const [primero, segundo, ...resto] = [1, 2, 3, 4, 5];
console.log(primero); // 1
console.log(segundo); // 2
console.log(resto); // [3, 4, 5]
```

## Ejemplos Extra

```js
//Comparaciones números
alert(2 > 1); // true (correcto)
alert(2 == 1); // false (incorrecto)
alert(2 != 1); // true (correcto)

//Comparaciones cadenas
/*
Compara por orden “de diccionario” o “lexicográfico”.
Si los primeros caracteres de ambas cadenas son los mismos, compare los segundos caracteres de la misma manera.
Si ambas cadenas tienen la misma longitud, entonces son iguales. De lo contrario, la cadena más larga es mayor.
Una letra mayúscula "A" no es igual a la minúscula "a". ¿Cuál es mayor? La "a" minúscula. ¿Por qué? Porque el carácter en minúsculas tiene un mayor índice en la tabla de codificación interna que utiliza JavaScript (Unicode).
*/
alert("Z" > "A"); // true
alert("Glow" > "Glee"); // true
alert("Bee" > "Be"); // true

//Igualdad estricta
alert(0 == false); // true
alert("" == false); // true
alert(0 === false); // falso, porque los tipos son diferentes

//El valor undefined no debe compararse con otros valores:
//Trata cualquier comparación con undefined/null (excepto la igualdad estricta ===) con sumo cuidado.
alert(null == undefined); // true
alert(null === undefined); // false
alert(undefined > 0); // false (1)
alert(undefined < 0); // false (2)
alert(undefined == 0); // false (3)
```
