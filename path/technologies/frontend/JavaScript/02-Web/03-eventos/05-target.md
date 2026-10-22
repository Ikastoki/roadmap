<h1 align="center">Propiedad target</h1>

<h2>📑 Contenido</h2>

- [Propiedad target](#propiedad-target)
- [event.target vs. event.currentTarget](#eventtarget-vs-eventcurrenttarget)

## Propiedad target

La propiedad target es un elemento clave del objeto de evento que te permite identificar con precisión dónde se originó un evento, sin importar dónde se esté manejando.

## event.target vs. event.currentTarget

La diferencia entre `target` y `currentTarget` es fundamental para entender la propagación de eventos.

- `event.target`: Es el elemento en el que se hizo clic o el que inició el evento. Siempre será el nodo más profundo en la jerarquía del DOM donde el evento ocurrió.

- `event.currentTarget`: Es el elemento al que se adjuntó el `event listener`. El valor de `currentTarget` cambia a medida que el evento viaja a través de la fase de captura o de burbujeo.

Ejemplo práctico

```html
<ul id="lista">
  <li class="item">Elemento 1</li>
  <li class="item">Elemento 2</li>
  <li class="item">Elemento 3</li>
</ul>
```

Si queremos saber en qué `<li>` específico hizo clic el usuario, podemos usar la propagación de eventos en nuestro beneficio. En lugar de adjuntar un event `listener` a cada `<li>` (lo que sería ineficiente), podemos adjuntar un único `listener` a su padre, el `<ul>`.

```javascript
document.getElementById("lista").addEventListener("click", function (e) {
  // e.target es el <li> en el que se hizo clic
  console.log("Hiciste clic en:", e.target);

  // e.currentTarget es el <ul>, que es donde se adjuntó el listener
  console.log("El listener está en:", e.currentTarget);
});
```

Al hacer clic en "Elemento 2", el evento se origina en ese `<li>`. Gracias a la propagación (burbujeo), el evento viaja hasta el `<ul>`, donde nuestro listener lo atrapa. En ese momento:

- `e.target` será el elemento `<li>` con el texto "Elemento 2".

- `e.currentTarget` será el elemento `<ul>` con el id="lista".

De esta forma, puedes usar `e.target` para realizar acciones específicas en el elemento que se activó, como cambiar su color o eliminarlo, sin tener que escribir código para cada elemento individual.
