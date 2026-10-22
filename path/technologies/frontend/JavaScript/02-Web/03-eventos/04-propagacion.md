<h1 align="center">Propagación de Eventos</h1>

<h2>📑 Contenido</h2>

- [Propagación de Eventos](#propagación-de-eventos)
- [Controlando la Propagación con addEventListener()](#controlando-la-propagación-con-addeventlistener)
- [Detener la Propagación (e.stopPropagation())](#detener-la-propagación-estoppropagation)

## Propagación de Eventos

Cuando un evento ocurre en un elemento del DOM (por ejemplo, un clic en un botón), ese evento no solo se queda en ese elemento. El evento se "propaga" a través de la jerarquía de nodos del DOM. Este viaje del evento tiene dos fases principales:

1. Fase de Captura (Capturing Phase): El evento viaja desde el nodo más alto de la jerarquía del DOM (`window`, `document`, `<html>`, `<body>`) hacia el elemento donde ocurrió el evento.

2. Fase de Burbujeo (Bubbling Phase): El evento viaja de vuelta, desde el elemento donde se originó, subiendo por la jerarquía del DOM hasta el nodo más alto.

El navegador ejecuta cualquier controlador de eventos que encuentre en su camino, tanto en la fase de captura como en la de burbujeo. Por defecto, `addEventListener()` escucha los eventos en la fase de burbujeo.

## Controlando la Propagación con addEventListener()

`addEventListener()` tiene un tercer parámetro opcional que te permite elegir en qué fase quieres escuchar el evento.

`elemento.addEventListener(evento, funcion, [opciones]);`

El parámetro `[opciones]` puede ser un objeto o un valor booleano. Si se establece en true, el controlador de eventos se activará en la fase de captura. Si se establece en false (por defecto), se activará en la fase de burbujeo.

```html
<div id="padre">
  <button id="hijo">Haz clic</button>
</div>
```

```javascript
const padre = document.getElementById("padre");
const hijo = document.getElementById("hijo");

// Escucha el evento en la fase de burbujeo (por defecto)
padre.addEventListener("click", function () {
  console.log("Fase de Burbujeo: Click en el Padre");
});

// Escucha el evento en la fase de captura
hijo.addEventListener(
  "click",
  function () {
    console.log("Fase de Captura: Click en el Hijo");
  },
  { capture: true }
);
```

Al hacer clic en el botón con `id="hijo"`, la secuencia de eventos en la consola sería:

1. **Fase de Captura: Click en el Hijo** (se activa primero el controlador de la fase de captura).

2. **Fase de Burbujeo: Click en el Padre** (el evento burbujea hacia arriba, activando el controlador del padre).

## Detener la Propagación (e.stopPropagation())

A veces, no quieres que un evento se propague más allá de un elemento. Para eso, utilizas el método `stopPropagation()` en el objeto del evento.

```javascript
hijo.addEventListener("click", function (e) {
  e.stopPropagation(); // Detiene el burbujeo
  console.log("Se detuvo la propagación");
});

padre.addEventListener("click", function (e) {
  console.log("El evento no llegó aquí");
});
```

En este caso, al hacer clic en el botón hijo, el evento se detendrá y no llegará al elemento padre.
