<h1 align="center">Templates</h1>

<h2>📑 Contenido</h2>

- [¿Qué son las etiquetas template?](#qué-son-las-etiquetas-template)
- [Utilidad](#utilidad)
- [Como se utilizan](#como-se-utilizan)

## ¿Qué son las etiquetas template?

Las etiquetas `<template>` en HTML son una forma de definir fragmentos de contenido que no se renderizan cuando la página se carga inicialmente, pero que pueden clonarse y renderizarse posteriormente mediante JavaScript. Estas etiquetas son útiles cuando deseas incluir contenido en tu página web que no debe mostrarse de inmediato, sino que puede generarse dinámicamente cuando sea necesario.

## Utilidad

Las etiquetas `<template>` son útiles en situaciones como:

- Crear componentes web reutilizables que pueden instanciarse varias veces en una página.
- Almacenar fragmentos de contenido que se pueden mostrar cuando se cumplen ciertas condiciones.
- Generar contenido dinámicamente desde datos obtenidos de una fuente externa, como una API, y mostrarlo en la página.

## Como se utilizan

1. Definición del Template: Dentro de una etiqueta `<template>`, puedes colocar cualquier contenido HTML, como elementos, texto y otros elementos HTML válidos.

```html
<!-- HTML -->
<template id="miTemplate">
  <p>Este es un ejemplo de template.</p>
</template>
```

2. Clonación y Renderización: Para mostrar el contenido de un template en la página, puedes clonar el contenido del template y luego adjuntarlo al DOM utilizando JavaScript.

```js
// JavaScript

// Clonar el contenido del template
const template = document.getElementById("miTemplate");
const clone = document.importNode(template.content, true);

// Adjuntar el contenido clonado al DOM
document.body.appendChild(clone);
```

> [!NOTE]
>
> Las etiquetas `<template>` son especialmente útil para crear páginas web interactivas y dinámicas.


