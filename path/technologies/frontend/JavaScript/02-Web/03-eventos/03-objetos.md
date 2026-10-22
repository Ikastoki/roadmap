<h1 align="center">Objetos de eventos</h1>

<h2>📑 Contenido</h2>

- [](#)

## ¿Qué es el Objeto de Evento (event)?

Cuando ocurre un evento (un clic, una pulsación de tecla, etc.), el navegador crea automáticamente un objeto de evento y lo pasa como el primer argumento a la función controladora de eventos. Este objeto es un contenedor de información que describe en detalle lo que acaba de suceder.

Acceder a este objeto es crucial para entender el evento y manipularlo. Tradicionalmente, se le nombra como `event`, `e` o `evt` por convención.

```javascript
document.addEventListener("click", function (e) {
  // `e` es el objeto de evento
  console.log(e);
});
```

## Propiedades Clave del Objeto de Evento

El objeto de evento contiene una gran cantidad de propiedades y métodos, pero estas son algunas de las más importantes:

- `event.type`: Devuelve el tipo de evento que ocurrió (ej. 'click', 'keydown', 'submit').

- `event.target`: Es la referencia al elemento DOM donde el evento se originó. Si haces clic en un `<button>` dentro de un `<div>`, e.target será el `<button>`.

- `event.currentTarget`: Es la referencia al elemento DOM al que se le adjuntó el controlador de eventos. Si tienes un controlador de eventos en el `<div>` y haces clic en el `<button>`, e.currentTarget será el `<div>`.

- `event.preventDefault()`: Un método esencial para detener la acción por defecto del navegador para un evento. Por ejemplo, evita que un enlace te redirija a otra página o que un formulario se envíe.

- `event.stopPropagation()`: Un método clave para controlar la propagación de eventos. Detiene el flujo del evento, evitando que llegue a los elementos padre (bubbling) o hijo (capturing).

Identificar el elemento clicado:

```html
<div id="contenedor">
  <button id="boton">Botón</button>
</div>
```

```javascript
document.getElementById("contenedor").addEventListener("click", function (e) {
  // e.target: el botón en el que se hizo clic
  console.log("Target:", e.target.id);

  // e.currentTarget: el div al que se adjuntó el evento
  console.log("CurrentTarget:", e.currentTarget.id);
});
```

En este caso, aunque haces clic en el botón, el evento se propaga al `<div>`, donde el controlador de eventos está escuchando. e.target te dice dónde empezó el evento, y e.currentTarget te dice dónde lo estás manejando.

Evitar la acción por defecto:

```html
<a href="https://google.com" id="enlace">Ir a Google</a>
```

```javascript
document.getElementById("enlace").addEventListener("click", function (e) {
  // Previene que el enlace te redirija a Google
  e.preventDefault();
  console.log("Acción por defecto prevenida");
});
```

Este es un uso muy común, por ejemplo, para manejar el envío de formularios con JavaScript en lugar de dejar que el navegador lo haga.
