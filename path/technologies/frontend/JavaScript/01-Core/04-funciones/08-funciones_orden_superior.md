<h1 align='center'>Funciones de Orden Superior</h1>

<h2>📑 Contenido</h2>

- [Funciones de Orden Superior](#funciones-de-orden-superior)
- [¿Qué es una Función de Orden Superior?](#qué-es-una-función-de-orden-superior)
- [Funciones como Argumentos](#funciones-como-argumentos)
- [Funciones que Devuelven Otras Funciones](#funciones-que-devuelven-otras-funciones)
- [Currying: Transformando Funciones](#currying-transformando-funciones)
- [Métodos de Array Basados en Funciones de Orden Superior](#métodos-de-array-basados-en-funciones-de-orden-superior)

## Funciones de Orden Superior

Las funciones de orden superior (HOF, por sus siglas en inglés: Higher-Order Functions) son un concepto clave en JavaScript y otros lenguajes de programación funcional. Estas funciones no solo son esenciales para un estilo de programación más conciso y expresivo, sino que también proporcionan una gran flexibilidad y poder al trabajar con funciones y datos. En este capítulo, exploraremos en detalle qué son las funciones de orden superior, cómo se usan, y cómo pueden mejorar la estructura y la legibilidad del código.

## ¿Qué es una Función de Orden Superior?

Una función de orden superior es una función que cumple con al menos una de las siguientes condiciones:

- Recibe una o más funciones como argumento.
- Devuelve otra función como resultado.

En esencia, las funciones de orden superior tratan a las funciones como valores, ya que en JavaScript las funciones son ciudadanos de primera clase. Esto significa que se pueden asignar a variables, pasar como argumentos a otras funciones, o devolver desde funciones.

```js
function procesar(operacion, a, b) {
  return operacion(a, b);
}

function sumar(x, y) {
  return x + y;
}

console.log(procesar(sumar, 5, 3)); // Imprime 8
```

En este ejemplo, procesar es una función de orden superior porque recibe una función (sumar) como argumento.

## Funciones como Argumentos

Uno de los casos más comunes de funciones de orden superior es pasar una función como argumento a otra función. Esto es útil en varios patrones de programación, como los callbacks, las promesas, y los eventos asíncronos.

```js
setTimeout(function () {
  console.log("Han pasado 2 segundos");
}, 2000);
```

Aquí, `setTimeout` recibe una función anónima como argumento, que será ejecutada después de 2 segundos.

## Funciones que Devuelven Otras Funciones

Otro patrón poderoso de las funciones de orden superior es que pueden devolver otras funciones. Esto es muy útil para crear funciones personalizadas dinámicamente o para aplicar técnicas de currying.

```js
function crearSaludo(saludo) {
  return function (nombre) {
    return saludo + ", " + nombre;
  };
}

let saludarEnEspañol = crearSaludo("Hola");
console.log(saludarEnEspañol("Carlos")); // Imprime "Hola, Carlos"
```

En este caso, crearSaludo es una función de orden superior porque devuelve una nueva función que combina el saludo y el nombre.

## Currying: Transformando Funciones

El currying es una técnica en la que una función con múltiples argumentos se transforma en una serie de funciones que toman uno o más argumentos a la vez. En lugar de recibir todos los parámetros de una vez, las funciones devuelven nuevas funciones hasta que se han proporcionado todos los argumentos

```js
function sumar(a) {
  return function (b) {
    return a + b;
  };
}

let sumaCinco = sumar(5);
console.log(sumaCinco(3)); // Imprime 8
```

En este ejemplo, sumar es una función curried que recibe el primer argumento a y devuelve una nueva función que espera el segundo argumento b.

## Métodos de Array Basados en Funciones de Orden Superior

JavaScript tiene varios métodos de array que se basan en funciones de orden superior. Estos métodos son fundamentales en la programación funcional y permiten manipular colecciones de datos de manera declarativa, lo que hace que el código sea más legible y fácil de mantener.

> [!NOTE]
> Veremos estos métodos (forEach, map, filter, reduce...) en la sección de arrays.
