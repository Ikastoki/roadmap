<h1 align="center">Polyfill</h1>

<h2>📑 Contenido</h2>

- [¿Qué es un Polyfill?](#qué-es-un-polyfill)
- [Ejemplo Práctico: Array.prototype.includes()](#ejemplo-práctico-arrayprototypeincludes)
- [Polyfill vs. Transpilación](#polyfill-vs-transpilación)

## ¿Qué es un Polyfill?

Un polyfill es un fragmento de código (generalmente JavaScript) que proporciona una funcionalidad moderna a los navegadores que no la tienen de forma nativa. Su nombre proviene de la analogía con el producto de la marca Polyfilla (o similar), que se usa para rellenar grietas en las paredes. De la misma forma, un polyfill "rellena" las grietas de compatibilidad entre navegadores.

El objetivo principal de un polyfill es permitirte escribir código utilizando las últimas características del lenguaje (ES6, ES7, etc.) con la tranquilidad de que funcionará en todos los navegadores que necesites soportar.

## Ejemplo Práctico: Array.prototype.includes()

La función Array.prototype.includes() es una característica de ES7 que verifica si un elemento está en un array.

Código moderno (ES7):

```javascript
const frutas = ["manzana", "pera", "uva"];
console.log(frutas.includes("pera")); // Salida: true
```

En un navegador moderno, este código funciona sin problemas. Sin embargo, en un navegador muy antiguo, frutas.includes podría ser undefined, lo que causaría un error.

Un polyfill para includes() podría verse así:

```javascript
// Si la función `includes` no existe en el prototipo de Array...
if (!Array.prototype.includes) {
  // ...la creamos
  Array.prototype.includes = function (elemento) {
    // Implementamos la lógica para buscar el elemento
    for (let i = 0; i < this.length; i++) {
      if (this[i] === elemento) {
        return true;
      }
    }
    return false;
  };
}

const frutas = ["manzana", "pera", "uva"];
console.log(frutas.includes("pera")); // Ahora funciona en navegadores antiguos
```

Con este código, el navegador primero verifica si includes() existe. Si no es así, lo crea, asegurando que tu código moderno no falle.

## Polyfill vs. Transpilación

Es común confundir los polyfills con la transpilación (que realiza herramientas como Babel). Aunque ambos buscan la compatibilidad entre navegadores, tienen funciones diferentes:

- Transpilador: Convierte el código moderno a una versión más antigua. Por ejemplo, transforma una función flecha () => {} en una función tradicional function() {}. No añade nuevas funcionalidades al navegador.

- Polyfill: No cambia tu código, sino que añade la funcionalidad que falta en el navegador. Por ejemplo, añade el método includes() si no existe.

La combinación de ambos es lo que permite un desarrollo moderno y compatible:

- Usas Babel para convertir la sintaxis moderna a ES5.

- Usas polyfills para añadir las funciones y objetos que faltan en ES5.
