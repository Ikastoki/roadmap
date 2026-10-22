<h1 align="center">Modulos</h1>

<h2>📑 Contenido</h2>

- [Módulos en JavaScript](#módulos-en-javascript)
- [CommonJS](#commonjs)
  - [Exportar](#exportar)
  - [Importar](#importar)
- [Módulos ES6 (ESM)](#módulos-es6-esm)
  - [Exportar](#exportar-1)
  - [Importar](#importar-1)

## Módulos en JavaScript

Antes de los módulos, todo el código JavaScript se escribía en archivos globales. Esto creaba problemas con el "alcance" (scope) de las variables y colisiones de nombres. Los módulos resuelven esto permitiendo dividir el código en archivos separados, cada uno con su propio alcance. Un módulo puede exportar funciones o variables para que otros módulos las importen y las usen

## CommonJS

CommonJS es el sistema de módulos original de Node.js. Es un enfoque síncrono que se diseñó para entornos del lado del servidor.

Sintaxis de CommonJS

### Exportar

Se usa la palabra clave module.exports para exportar valores desde un módulo. Puedes exportar un solo valor (un objeto, una función, etc.) o un objeto con múltiples propiedades.

```javascript
// archivo: math.js
const sumar = (a, b) => a + b;
const restar = (a, b) => a - b;

module.exports = {
  sumar,
  restar,
};
```

### Importar

Se usa la función `require()` para importar un módulo.

```javascript
// archivo: app.js
const { sumar, restar } = require("./math.js");

console.log(sumar(5, 3)); // Salida: 8
```

Cuando llamas a `require()`, Node.js carga y ejecuta el archivo de forma síncrona, y la función devuelve el objeto que fue exportado.

## Módulos ES6 (ESM)

ES6 (ECMAScript 2015) introdujo un sistema de módulos nativo para JavaScript, diseñado para ser estándar tanto en el navegador como en el servidor. Es un enfoque asíncrono y más estático que CommonJS.

Sintaxis de ES6 Modules

### Exportar

Se usan las palabras clave `export` y `export default`.

- `export`: Para exportar múltiples valores con nombre.

- `export default`: Para exportar un valor principal del módulo.

```javascript
// archivo: math.js
export const sumar = (a, b) => a + b;

const restar = (a, b) => a - b;
export default restar;
```

### Importar

Se usa la palabra clave import.

```javascript
// archivo: app.js
import { sumar } from "./math.js";
import restar from "./math.js"; // Importa el valor por defecto

console.log(sumar(5, 3)); // Salida: 8
console.log(restar(5, 3)); // Salida: 2
```

Diferencias clave

| Característica | CommonJS                          | ES Modules (ESM)                                                     |
| -------------- | --------------------------------- | -------------------------------------------------------------------- |
| Sintaxis       | `require()` / `module.exports`    | `import` / `export`                                                  |
| Naturaleza     | Síncrono                          | Asíncrono                                                            |
| Uso en Node.js | Por defecto en versiones antiguas | Disponible desde Node.js 13 con `"type": "module"` en `package.json` |

En el ecosistema de Node.js, CommonJS sigue siendo el sistema de módulos por defecto y el más común en proyectos legados o en librerías de terceros. Sin embargo, los ES Modules son el estándar moderno y son cada vez más utilizados.

Para usar módulos ES6 en Node.js, debes configurar tu package.json de la siguiente manera:

```json
{
  "type": "module",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

Si no se especifica `"type": "module"`, Node.js asume que los archivos `.js` usan CommonJS. También puedes usar la extensión de archivo `.mjs` para indicar explícitamente que un archivo es un módulo ES6.
