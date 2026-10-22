<h1 align='center'>Operaciones "clásicas"</h1>

<h2>📑 Contenido</h2>

- [Operaciones Más Usadas del DOM](#operaciones-más-usadas-del-dom)
  - [¿Qué es el DOM?](#qué-es-el-dom)
  - [Selección de Elementos](#selección-de-elementos)
  - [Manipulación de Contenido](#manipulación-de-contenido)
  - [Manipulación de Atributos](#manipulación-de-atributos)
  - [Manipulación de Estilos](#manipulación-de-estilos)
  - [Manipulación de Clases](#manipulación-de-clases)
  - [Crear y Eliminar Elementos](#crear-y-eliminar-elementos)
  - [Eventos](#eventos)
  - [Recorrer el DOM](#recorrer-el-dom)
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

---

## Operaciones Más Usadas del DOM

### ¿Qué es el DOM?

El **DOM (Document Object Model)** es una representación estructurada del contenido HTML de una página web. Permite acceder y modificar elementos, atributos, estilos y eventos desde JavaScript.

---

### Selección de Elementos

```javascript
document.getElementById("miId");
document.getElementsByClassName("miClase");
document.getElementsByTagName("div");
document.querySelector(".miClase");
document.querySelectorAll("p.miClase");
```

---

### Manipulación de Contenido

```javascript
element.innerText = "Texto visible";
element.textContent = "Texto sin formato";
element.innerHTML = "<strong>HTML embebido</strong>";
```

---

### Manipulación de Atributos

```javascript
element.setAttribute("href", "https://google.com");
let valor = element.getAttribute("href");
element.removeAttribute("disabled");

input.value = "Hola";
imagen.src = "imagen.jpg";
checkbox.checked = true;
```

---

### Manipulación de Estilos

```javascript
element.style.color = "red";
element.style.backgroundColor = "blue";
```

---

### Manipulación de Clases

```javascript
element.classList.add("nuevaClase");
element.classList.remove("otraClase");
element.classList.toggle("modoOscuro");
element.classList.contains("activa"); // true / false
```

---

### Crear y Eliminar Elementos

```javascript
let nuevoDiv = document.createElement("div");
nuevoDiv.textContent = "Hola mundo";
document.body.appendChild(nuevoDiv);
document.body.insertBefore(nuevoDiv, referencia);
element.remove();
```

---

### Eventos

```javascript
element.addEventListener("click", function () {
  alert("¡Clic!");
});
```

Eventos comunes: `click`, `mouseover`, `keydown`, `submit`, `input`.

---

### Recorrer el DOM

```javascript
element.parentNode;
element.children;
element.firstElementChild;
element.lastElementChild;
element.nextElementSibling;
element.previousElementSibling;
```

---

## DOM CSS

El **DOM CSS** permite a los desarrolladores interactuar y manipular los estilos CSS aplicados a los elementos de una página web mediante JavaScript.

---

### Editar y Modificar

#### Modificar Estilos en Línea

```javascript
elemento.style.color = "red";
elemento.style.backgroundColor = "blue";
elemento.style.fontSize = "20px";
```

```javascript
elemento.style.cssText =
  "color: white; background-color: black; padding: 10px;";
```

#### Clases CSS

```javascript
element.classList.add("nueva-clase");
element.classList.remove("vieja-clase");
element.classList.toggle("activo");
```

---

### Crear y Añadir

#### Crear Clases Dinámicamente

```javascript
elemento.className = "clase1 clase2";
```

#### Añadir Estilos mediante una Hoja de Estilos Dinámica

```javascript
let estilo = document.createElement("style");
estilo.type = "text/css";
estilo.innerHTML =
  ".miClaseDinamica { color: green; background-color: yellow; }";
document.getElementsByTagName("head")[0].appendChild(estilo);
```

---

### Más Métodos

#### getComputedStyle()

```javascript
let estilos = window.getComputedStyle(elemento);
console.log(estilos.color);
console.log(estilos.margin);
```

#### Otras Utilidades

```javascript
let hojas = document.styleSheets;
document.styleSheets[0].insertRule(".nueva-regla { font-weight: bold; }", 0);
document.styleSheets[0].deleteRule(0);
```
