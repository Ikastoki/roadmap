<h1 align='center'>Nodos</h1>

<h2>📑 Contenido</h2>

- [Nodos](#nodos)
- [Tipos de Nodos](#tipos-de-nodos)
- [Traversing DOM](#traversing-dom)
  - [Propiedades de Traversing](#propiedades-de-traversing)
  - [Ejemplo de Traversing](#ejemplo-de-traversing)
- [NodeList](#nodelist)
- [childNodes](#childnodes)
  - [Notas sobre childNodes](#notas-sobre-childnodes)

## Nodos

En el contexto del **Document Object Model (DOM)**, un **nodo** es una unidad fundamental de la estructura de un documento. Cada parte de un documento HTML o XML es representada como un nodo en el DOM, que puede ser un nodo de elemento, nodo de texto, nodo de atributo, entre otros. La manipulación de estos nodos es crucial para crear aplicaciones web dinámicas.

## Tipos de Nodos

Los tipos más comunes de nodos en el DOM son:

- **Nodos de Elemento**: Representan elementos HTML, como `<div>`, `<p>`, `<span>`, etc.
- **Nodos de Texto**: Representan el contenido textual dentro de los nodos de elemento.
- **Nodos de Comentario**: Representan comentarios dentro del documento.
- **Nodos de Documento**: Representan el documento en sí mismo.

## Traversing DOM

**Traversing DOM** se refiere al proceso de navegar a través de los nodos del DOM. Esto permite a los desarrolladores acceder a nodos específicos y manipular su contenido o atributos. Hay varias propiedades y métodos que facilitan este recorrido.

### Propiedades de Traversing

- **parentNode**: Devuelve el nodo padre de un nodo específico.
- **firstChild**: Devuelve el primer hijo de un nodo.
- **lastChild**: Devuelve el último hijo de un nodo.
- **nextSibling**: Devuelve el siguiente hermano del nodo.
- **previousSibling**: Devuelve el hermano anterior del nodo.

### Ejemplo de Traversing

```javascript
let elemento = document.getElementById("miElemento");
let padre = elemento.parentNode; // Obtiene el nodo padre
let primerHijo = padre.firstChild; // Obtiene el primer hijo
let siguienteHermano = elemento.nextSibling; // Obtiene el siguiente hermano
```

## NodeList

Un NodeList es una colección de nodos devuelta por ciertos métodos del DOM, como `querySelectorAll(), childNodes y getElementsByClassName()`. A diferencia de los arrays, los NodeLists no tienen métodos como `map()`, `forEach()`, etc., pero se pueden recorrer con un bucle `for`.

```js
let nodos = document.querySelectorAll("p"); // Obtiene todos los párrafos
nodos.forEach(function (nodo) {
  console.log(nodo.innerHTML); // Muestra el contenido de cada párrafo
});
```

## childNodes

La propiedad childNodes devuelve una colección (NodeList) de todos los nodos hijos de un nodo específico, incluyendo nodos de texto, elementos y comentarios. Esta colección puede incluir nodos que no son elementos, como nodos de texto.

```js
let contenedor = document.getElementById("miContenedor");
let hijos = contenedor.childNodes; // Obtiene todos los nodos hijos

for (let i = 0; i < hijos.length; i++) {
  console.log(hijos[i].nodeName); // Muestra el nombre de cada nodo hijo
}
```

### Notas sobre childNodes

L- a propiedad `childNodes` incluye todos los tipos de nodos, no solo elementos. Por lo tanto, es común que la colección contenga nodos de texto (por ejemplo, espacios en blanco entre etiquetas).

- Para obtener solo los nodos de elementos hijos, se puede utilizar la propiedad `children`, que devuelve una colección HTMLCollection de nodos de elemento.
