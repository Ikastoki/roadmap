<h1 align="center">Delegación de eventos</h1>

<h2>📑 Contenido</h2>

- [¿Qué es la Delegación de Eventos?](#qué-es-la-delegación-de-eventos)
- [¿Por qué usar la Delegación de Eventos?](#por-qué-usar-la-delegación-de-eventos)
- [Ejemplo Práctico](#ejemplo-práctico)
  - [Sin Delegación (ineficiente):](#sin-delegación-ineficiente)
  - [Con Delegación (eficiente):](#con-delegación-eficiente)

## ¿Qué es la Delegación de Eventos?

La delegación de eventos es el patrón de diseño que consiste en adjuntar un único `event listener` a un elemento padre en lugar de adjuntar un `listener` a cada uno de sus elementos hijos.

Esta técnica aprovecha el fenómeno de la propagación de eventos (el burbujeo, por defecto). Cuando un evento se dispara en un elemento hijo, este "burbujea" hacia arriba a través de la jerarquía del DOM, llegando al elemento padre. El `event listener` del padre captura el evento y, gracias al objeto de evento (`e.target`), puede identificar exactamente en qué elemento hijo se originó la acción.

## ¿Por qué usar la Delegación de Eventos?

- Rendimiento: Es mucho más eficiente adjuntar un solo `event listener` que adjuntar cientos o miles. Esto reduce el consumo de memoria y mejora el rendimiento, especialmente en páginas con listas de elementos muy largas.

- Código Dinámico: Es la solución perfecta para manejar elementos que se añaden dinámicamente al DOM. Si añades nuevos elementos a una lista, el `event listener` en el padre ya está listo para manejarlos sin necesidad de adjuntarles un nuevo `listener` individualmente.

- Simplicidad: El código es más limpio y fácil de mantener.

## Ejemplo Práctico

Imagina que tienes una lista de tareas y quieres que, al hacer clic en cualquier tarea, esta se marque como completada.

### Sin Delegación (ineficiente):

```html
<ul id="lista-tareas">
  <li>Tarea 1</li>
  <li>Tarea 2</li>
  <li>Tarea 3</li>
</ul>
```

```javascript
const tareas = document.querySelectorAll("#lista-tareas li");

tareas.forEach((tarea) => {
  // Se añade un listener a CADA elemento, lo que es ineficiente
  tarea.addEventListener("click", function () {
    tarea.classList.toggle("completada");
  });
});
```

### Con Delegación (eficiente):

```html
<ul id="lista-tareas">
  <li>Tarea 1</li>
  <li>Tarea 2</li>
  <li>Tarea 3</li>
</ul>
```

```javascript
document.getElementById("lista-tareas").addEventListener("click", function (e) {
  // Comprobamos que el elemento clicado sea un `li`
  if (e.target.tagName === "LI") {
    e.target.classList.toggle("completada");
  }
});
```

En este ejemplo, solo tenemos un listener en el `<ul>`. Cuando haces clic en cualquier `<li>` hijo, el evento burbujea hasta el `<ul>`. La condición `e.target.tagName === 'LI'` nos permite asegurarnos de que la acción solo se ejecute si se hizo clic en un elemento de la lista y no en el propio `<ul>`.

Esta es la forma más limpia y eficiente de manejar eventos en elementos hijos de un contenedor.
