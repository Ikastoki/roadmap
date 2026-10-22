<h1 align='center'>Métodos para cadenas de texto que soportan RegExp</h1>

<h2>📑 Contenido</h2>

- [Métodos para cadenas de texto que soportan RegExp](#métodos-para-cadenas-de-texto-que-soportan-regexp)
  - [match()](#match)
  - [matchAll()](#matchall)
  - [replace()](#replace)
  - [replaceAll()](#replaceall)
  - [search()](#search)
  - [split()](#split)
- [Métodos del Objeto RegExp](#métodos-del-objeto-regexp)
  - [test()](#test)
  - [exec()](#exec)
- [Métodos avanzados de manejo de expresiones regulares](#métodos-avanzados-de-manejo-de-expresiones-regulares)
  - [Propiedad lastIndex](#propiedad-lastindex)

## Métodos para cadenas de texto que soportan RegExp

JavaScript proporciona varios métodos para trabajar con expresiones regulares (RegExp), tanto en las cadenas de texto como en los propios objetos RegExp. Estos métodos permiten realizar búsquedas, coincidencias y reemplazos en las cadenas de manera eficiente. En este capítulo, exploraremos los métodos clave asociados con las expresiones regulares y sus aplicaciones más comunes.

JavaScript ofrece varios métodos de los objetos String que aceptan expresiones regulares como argumentos para realizar tareas de búsqueda, coincidencia y manipulación de cadenas. Algunos de los métodos más útiles son:

### match()

El método `match()` busca coincidencias entre una cadena y una expresión regular. Devuelve un array con las coincidencias encontradas o null si no se encuentran coincidencias.

Si la expresión regular tiene la bandera g (global), devuelve todas las coincidencias en un array.
Sin la bandera g, devuelve solo la primera coincidencia junto con detalles adicionales sobre la misma.

```js
let texto = "Los gatos son grandes. Los perros son pequeños.";
let regex = /Los/g;
console.log(texto.match(regex)); // ["Los", "Los"]
```

### matchAll()

El método `matchAll()` es similar a `match()`, pero siempre devuelve todas las coincidencias junto con los detalles (grupos de captura), incluso cuando la expresión regular no tiene la bandera `g`. Este método devuelve un iterador, que puede ser convertido en un array con `Array.from()`.

```js
let texto = "1 gato, 2 perros, 3 elefantes";
let regex = /\d+ (\w+)/g; // Captura el número seguido de una palabra
let coincidencias = texto.matchAll(regex);

for (const match of coincidencias) {
  console.log(match);
}
// ["1 gato", "gato"]
// ["2 perros", "perros"]
// ["3 elefantes", "elefantes"]
```

### replace()

El método `replace()` busca coincidencias en una cadena usando una expresión regular o una subcadena literal y reemplaza las coincidencias con un nuevo valor. Puede realizar un reemplazo global con la bandera g o reemplazar solo la primera coincidencia.

```js
let texto = "Perro y Gato";
let nuevoTexto = texto.replace(/Gato/, "Elefante");
console.log(nuevoTexto); // "Perro y Elefante"
```

Con una función de reemplazo:

```js
let texto = "1 perro, 2 gatos";
let nuevoTexto = texto.replace(/(\d+)/g, (match, p1) => {
  return parseInt(p1) * 2;
});
console.log(nuevoTexto); // "2 perro, 4 gatos"
```

### replaceAll()

El método `replaceAll()` es similar a `replace()`, pero reemplaza todas las coincidencias sin necesidad de usar la bandera `g`. Se utiliza con literales o expresiones regulares.

```js
let texto = "gato gato gato";
let nuevoTexto = texto.replaceAll("gato", "perro");
console.log(nuevoTexto); // "perro perro perro"
```

### search()

El método `search()` busca una coincidencia de la expresión regular en la cadena. Devuelve el índice de la primera coincidencia o `-1` si no encuentra ninguna.

```js
let texto = "Los gatos son inteligentes.";
let indice = texto.search(/gatos/);
console.log(indice); // 4
```

### split()

El método `split()` divide una cadena en un array de subcadenas utilizando una expresión regular o una subcadena como delimitador. Si se usa una expresión regular con grupos de captura, las subcadenas capturadas también se incluirán en el array resultante.

```js
let texto = "manzana, naranja, plátano";
let frutas = texto.split(/,\s*/);
console.log(frutas); // ["manzana", "naranja", "plátano"]
```

## Métodos del Objeto RegExp

Además de los métodos en las cadenas, los objetos RegExp en JavaScript tienen sus propios métodos para realizar y optimizar las búsquedas.

### test()

El método `test()` evalúa si una expresión regular tiene coincidencias dentro de una cadena. Devuelve true si encuentra una coincidencia, y false si no.

```js
let regex = /\d+/; // Coincide con uno o más dígitos
console.log(regex.test("123")); // true
console.log(regex.test("abc")); // false
```

### exec()

El método `exec()` busca coincidencias en una cadena y devuelve detalles adicionales. Devuelve un array con la primera coincidencia y los grupos capturados, o null si no encuentra coincidencias. Este método es útil cuando necesitas detalles adicionales sobre las coincidencias.

Si se usa con la bandera `g`, `exec()` puede usarse repetidamente para encontrar todas las coincidencias, manteniendo un índice interno de búsqueda.

```js
let regex = /(\d+) perro/;
let resultado = regex.exec("3 perros en el parque");

console.log(resultado[0]); // "3 perro"
console.log(resultado[1]); // "3" (grupo capturado)
console.log(resultado.index); // 0 (posición de la coincidencia)
```

## Métodos avanzados de manejo de expresiones regulares

Existen otros métodos y características en JavaScript para trabajar de manera más avanzada con expresiones regulares, como:

### Propiedad lastIndex

La propiedad lastIndex en un objeto RegExp indica el punto de partida de la siguiente búsqueda en cadenas con el método `exec()` cuando se usa la bandera `g`. Permite buscar múltiples coincidencias de forma secuencial.

```js
let regex = /\d+/g; // Busca todos los números
let texto = "1 perro, 2 gatos, 3 elefantes";

let resultado;
while ((resultado = regex.exec(texto)) !== null) {
  console.log(`Encontrado: ${resultado[0]} en el índice ${resultado.index}`);
}
```
