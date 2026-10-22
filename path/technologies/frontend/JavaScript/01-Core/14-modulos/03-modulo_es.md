<h1 align='center'>Módulos ES (ECMAScript)</h1>

<h2>📑 Contenido</h2>

- [Módulos ES (ECMAScript)](#módulos-es-ecmascript)
- [¿Qué son los Módulos ES?](#qué-son-los-módulos-es)
  - [Ventajas de los Módulos ES:](#ventajas-de-los-módulos-es)
- [Exportaciones en los Módulos ES](#exportaciones-en-los-módulos-es)
  - [Exportación Nombrada (Named Export)](#exportación-nombrada-named-export)
  - [Exportación por defecto (Default Export)](#exportación-por-defecto-default-export)
- [Importaciones en los Módulos ES](#importaciones-en-los-módulos-es)
  - [Importación de exportaciones nombradas](#importación-de-exportaciones-nombradas)
  - [Importación de la exportación por defecto](#importación-de-la-exportación-por-defecto)
- [Mezclando Exportaciones Nombradas y por Defecto](#mezclando-exportaciones-nombradas-y-por-defecto)
- [Carga de Módulos ES en el Navegador](#carga-de-módulos-es-en-el-navegador)
- [Carga Diferida y Optimización](#carga-diferida-y-optimización)
- [Herramientas de Construcción y Empaquetado](#herramientas-de-construcción-y-empaquetado)

## Módulos ES (ECMAScript)

Con la llegada de ECMAScript 6 (ES6) en 2015, JavaScript se enriqueció con una nueva característica que revolucionó la forma en que se organizan y estructuran las aplicaciones: los Módulos ES. Antes de ES6, los desarrolladores utilizaban bibliotecas y soluciones externas como CommonJS (utilizado por Node.js) o AMD (Asynchronous Module Definition) para dividir el código en módulos. La especificación ES6 introdujo un sistema de módulos nativo, simplificando el proceso y brindando una solución estándar.

Los Módulos ES permiten a los desarrolladores exportar e importar código entre archivos de una manera limpia, eficiente y organizada. En este capítulo exploraremos cómo funcionan, cómo se utilizan y por qué son esenciales para el desarrollo moderno en JavaScript

## ¿Qué son los Módulos ES?

Un Módulo ES (ECMAScript Module) es simplemente un archivo de JavaScript que puede importar y exportar piezas de código desde otros módulos. Estos archivos operan de forma independiente, y su propósito es encapsular funciones, clases, objetos o valores para que puedan ser reutilizados o compartidos entre diferentes partes de una aplicación.

En términos simples, cada archivo de JavaScript en ES6 se puede considerar como un módulo, y podemos elegir qué partes del código hacer disponibles (exportar) y cuáles podemos tomar de otros módulos (importar).

### Ventajas de los Módulos ES:

- **Encapsulación:** Cada módulo tiene su propio espacio de nombres (scope), por lo que las variables y funciones definidas en un módulo no afectan a otros.
- **Reutilización:** Los módulos permiten definir componentes reutilizables que pueden ser utilizados en varias partes de una aplicación.
- **Carga perezosa (Lazy loading):** Los módulos pueden ser cargados de manera diferida, optimizando el rendimiento de la aplicación.
- **Optimización:** Las herramientas de bundling como Webpack o Rollup pueden optimizar automáticamente los módulos, eliminando el código no utilizado (tree-shaking) y mejorando el rendimiento de la aplicación.

## Exportaciones en los Módulos ES

Para que un módulo sea útil para otros archivos, debe exportar el código que desea compartir. JavaScript ofrece dos tipos de exportaciones en los Módulos ES:

### Exportación Nombrada (Named Export)

Este tipo de exportación permite exportar múltiples variables, funciones u objetos del mismo módulo, y cada uno de ellos debe ser importado utilizando su nombre original.

```js
// archivo: mathModule.js

// Exportando una función
export function sumar(a, b) {
  return a + b;
}

// Exportando una constante
export const PI = 3.14159;

// Exportando una clase
export class Calculadora {
  constructor() {
    this.resultado = 0;
  }
  sumar(a, b) {
    this.resultado = a + b;
    return this.resultado;
  }
}
```

En este ejemplo, estamos exportando una función sumar, una constante PI, y una clase Calculadora desde el módulo mathModule.js.

### Exportación por defecto (Default Export)

Cada módulo puede tener una única exportación por defecto. Este tipo de exportación es útil cuando deseas exportar un solo valor, clase o función como el "valor principal" del módulo.

```js
// archivo: greetingModule.js

// Exportando por defecto una función
export default function saludar(nombre) {
  return `Hola, ${nombre}!`;
}
```

En este ejemplo, saludar es la exportación por defecto del módulo greetingModule.js. No es necesario que la exportación por defecto tenga un nombre en el archivo de importación.

## Importaciones en los Módulos ES

Una vez que un módulo ha exportado algo, otro módulo puede importarlo para utilizarlo. Dependiendo de si el módulo exporta varias cosas (exportación nombrada) o solo una cosa (exportación por defecto), la sintaxis de importación varía.

### Importación de exportaciones nombradas

Cuando importas algo que ha sido exportado con su nombre, debes usar llaves `{}` y asegurarte de que los nombres coincidan con los nombres de las exportaciones.

```js
// archivo: main.js

// Importando funciones y constantes del módulo mathModule.js
import { sumar, PI, Calculadora } from "./mathModule.js";

console.log(sumar(2, 3)); // 5
console.log(PI); // 3.14159

const calc = new Calculadora();
console.log(calc.sumar(10, 20)); // 30
```

En este ejemplo, hemos importado varias cosas desde mathModule.js utilizando la sintaxis de importación nombrada.

### Importación de la exportación por defecto

Cuando un módulo tiene una exportación por defecto, se puede importar sin usar llaves `{}`, y puedes nombrarlo como quieras en el archivo de importación.

```js
// archivo: main.js

// Importando la función saludar desde greetingModule.js
import saludar from "./greetingModule.js";

console.log(saludar("Ana")); // "Hola, Ana!"
```

Aquí, hemos importado la función saludar como la exportación por defecto del archivo greetingModule.js.

## Mezclando Exportaciones Nombradas y por Defecto

Es posible que un mismo módulo tenga tanto exportaciones nombradas como una exportación por defecto. En este caso, puedes importar ambos tipos de exportación en un mismo archivo.

```js
// archivo: utilModule.js

export function sumar(a, b) {
  return a + b;
}

export function restar(a, b) {
  return a - b;
}

export default function multiplicar(a, b) {
  return a * b;
}
```

En este caso, puedes importar ambos tipos de exportación así:

```js
// archivo: main.js

import multiplicar, { sumar, restar } from "./utilModule.js";

console.log(multiplicar(3, 4)); // 12
console.log(sumar(10, 5)); // 15
console.log(restar(9, 3)); // 6
```

Aquí importamos multiplicar (la exportación por defecto) y sumar y restar (las exportaciones nombradas) desde utilModule.js.

## Carga de Módulos ES en el Navegador

Antes de ES6, para cargar scripts en el navegador, se utilizaba la etiqueta `<script>`. Sin embargo, esto no permitía la carga modular ni la reutilización sencilla de código entre archivos.

Con ES6, los módulos se cargan usando la etiqueta `<script>` con el atributo `type="module"`, que permite al navegador reconocer que está tratando con un módulo ES.

```html
<!-- archivo: index.html -->
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Ejemplo de Módulos ES</title>
  </head>
  <body>
    <script type="module" src="main.js"></script>
  </body>
</html>
```

Al usar `type="module"`, el navegador cargará el archivo `main.js` como un módulo. Esto tiene algunas implicaciones importantes:

- Los módulos se ejecutan en modo estricto (strict mode) por defecto.
- Las variables definidas en un módulo no son accesibles globalmente, lo que ayuda a prevenir la contaminación del espacio global de nombres.
- Los módulos son cargados de manera asíncrona, lo que significa que el navegador no detiene el procesamiento de la página mientras los carga.

## Carga Diferida y Optimización

Otra característica importante de los Módulos ES es que pueden ser cargados de manera diferida, lo que mejora el rendimiento de las aplicaciones. Esto se puede lograr añadiendo el atributo `defer` a la etiqueta `<script>`:

```js
<script type="module" src="main.js" defer></script>
```

Al usar `defer`, el navegador cargará el módulo sin bloquear la renderización del contenido HTML, lo que puede hacer que las aplicaciones se sientan más rápidas y responsivas.

## Herramientas de Construcción y Empaquetado

En aplicaciones grandes, es común utilizar herramientas como `Webpack, Rollup, o Parcel` para gestionar los módulos ES y empaquetar (bundle) todo el código en archivos más pequeños y optimizados para la web. Estas herramientas permiten:

- **Tree-shaking:** Eliminan el código que no se utiliza, reduciendo el tamaño de los archivos finales.
- **Minificación:** Comprimen el código eliminando espacios, comentarios y reduciendo el tamaño de las variables.
- **Optimización del rendimiento:** Generan módulos optimizados que se pueden cargar de manera más eficiente en el navegador.
