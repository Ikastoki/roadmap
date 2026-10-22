<h1 align='center'>DOM CSS</h1>

<h2>📑 Contenido</h2>

- [DOM CSS](#dom-css)
- [Editar y Modificar](#editar-y-modificar)
  - [Modificar Estilos en Línea](#modificar-estilos-en-línea)
  - [Clases CSS](#clases-css)
- [Crear y Añadir](#crear-y-añadir)
  - [Crear Clases Dinámicamente](#crear-clases-dinámicamente)
  - [Añadir Estilos mediante una Hoja de Estilos Dinámica](#añadir-estilos-mediante-una-hoja-de-estilos-dinámica)
- [Más Métodos](#más-métodos)
  - [getComputedStyle()](#getcomputedstyle)
- [Otras Utilidades](#otras-utilidades)
  - [styleSheets](#stylesheets)
  - [insertRule()](#insertrule)
  - [deleteRule()](#deleterule)

## DOM CSS

El **DOM CSS** permite a los desarrolladores interactuar y manipular los estilos CSS aplicados a los elementos de una página web mediante JavaScript. Esto incluye la posibilidad de editar, modificar, crear y añadir nuevos estilos, así como acceder y gestionar los atributos relacionados con la presentación visual de los elementos.

## Editar y Modificar

Existen diversas formas de modificar los estilos CSS de un elemento mediante JavaScript. La manera más directa es a través de la propiedad **style** de un elemento, que permite aplicar o modificar estilos en línea.

### Modificar Estilos en Línea

- **element.style**: Permite acceder y cambiar las propiedades CSS en línea de un elemento.

  ```javascript
  let elemento = document.getElementById("miElemento");
  elemento.style.color = "red"; // Cambia el color del texto
  elemento.style.backgroundColor = "blue"; // Cambia el color de fondo
  elemento.style.fontSize = "20px"; // Cambia el tamaño de la fuente
  ```

- **element.style.cssText**: Permite establecer múltiples estilos como una cadena de texto CSS.

  ```javascript
  elemento.style.cssText =
    "color: white; background-color: black; padding: 10px;";
  ```

### Clases CSS

En lugar de modificar los estilos directamente, a menudo es más eficiente cambiar o añadir clases CSS que ya tengan estilos definidos en una hoja de estilos.

- **element.classList.add()**: Añade una clase CSS a un elemento.

  ```javascript
  elemento.classList.add("nueva-clase");
  ```

- **element.classList.remove()**: Elimina una clase CSS de un elemento.

  ```javascript
  elemento.classList.remove("vieja-clase");
  ```

- **element.classList.toggle()**: Alterna la presencia de una clase CSS en un elemento (la añade si no está presente, o la elimina si ya está aplicada).

  ```javascript
  elemento.classList.toggle("activo");
  ```

---

## Crear y Añadir

En lugar de modificar estilos individuales, es posible crear y añadir nuevos estilos a un elemento utilizando las clases CSS.

### Crear Clases Dinámicamente

- **element.className**: Permite establecer una lista de clases para un elemento. Sobrescribe todas las clases existentes.

  ```javascript
  elemento.className = "clase1 clase2"; // Asigna múltiples clases al elemento
  ```

### Añadir Estilos mediante una Hoja de Estilos Dinámica

Es posible crear una nueva hoja de estilos utilizando JavaScript y añadirla al documento.

```javascript
let estilo = document.createElement("style");
estilo.type = "text/css";
estilo.innerHTML =
  ".miClaseDinamica { color: green; background-color: yellow; }";
document.getElementsByTagName("head")[0].appendChild(estilo);
```

Esto añadirá una nueva clase llamada miClaseDinamica a la hoja de estilos de la página.

## Más Métodos

Además de los métodos comunes como `style` y `classList`, existen otros métodos y propiedades útiles para manipular los estilos CSS mediante el DOM.

### getComputedStyle()

getComputedStyle(elemento) devuelve un objeto CSSStyleDeclaration que contiene los estilos calculados de un elemento. Estos estilos incluyen los valores finales aplicados después de que se hayan procesado todas las hojas de estilo (incluso las heredadas y las de los navegadores).

```javascript
let elemento = document.getElementById("miElemento");
let estilos = window.getComputedStyle(elemento);

console.log(estilos.color); // Muestra el color computado (ej. "rgb(255, 0, 0)")
console.log(estilos.margin); // Muestra el margen computado
```

Esto es útil cuando necesitas saber qué estilo visual está siendo aplicado realmente a un elemento en la página.

## Otras Utilidades

### styleSheets

El objeto document.styleSheets permite acceder a las hojas de estilo del documento, tanto internas como externas.

```javascript
let hojasDeEstilo = document.styleSheets;
console.log(hojasDeEstilo);
```

Se puede usar para inspeccionar, modificar o incluso crear nuevas reglas CSS dinámicamente.

### insertRule()

Permite añadir una nueva regla CSS a una hoja de estilo existente.

```javascript
document.styleSheets[0].insertRule(
  ".nueva-regla { font-weight: bold; }",
  document.styleSheets[0].cssRules.length
);
```

### deleteRule()

Permite eliminar una regla CSS de una hoja de estilo.

```javascript
document.styleSheets[0].deleteRule(0); // Elimina la primera regla
```
