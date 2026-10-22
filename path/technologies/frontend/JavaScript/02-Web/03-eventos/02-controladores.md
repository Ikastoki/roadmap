<h1 align="center">Controladores de Eventos</h1>

<h2>📑 Contenido</h2>

- [Controladores de Eventos (Event Handlers)](#controladores-de-eventos-event-handlers)
- [Atributos en HTML (onclick, onmouseover, etc.)](#atributos-en-html-onclick-onmouseover-etc)
- [addEventListener() (El método recomendado)](#addeventlistener-el-método-recomendado)

## Controladores de Eventos (Event Handlers)

Un controlador de eventos es una función de JavaScript que se ejecuta cuando ocurre un evento específico. Es el código que "maneja" el evento. Hay varias formas de asignar un controlador de eventos a un elemento, pero addEventListener() es la más recomendada y flexible.

## Atributos en HTML (onclick, onmouseover, etc.)

Esta es la forma más antigua y menos recomendada para asignar un controlador de eventos. Consiste en añadir el código de JavaScript directamente en un atributo del elemento HTML.

- Ventajas: Es muy simple para efectos puntuales y rápidos.

Desventajas:

- Acoplamiento: Mezcla la lógica de JavaScript con la estructura HTML, lo que dificulta la lectura y el mantenimiento.

- Limitaciones: Solo se puede asignar una función por evento a un elemento. Si intentas asignar otra, la sobrescribirás.

```javascript
const miBoton = document.getElementById("miBoton");

// Se asigna la función al controlador de evento onclick
miBoton.onclick = function () {
  console.log("Primera función");
};

// Esta segunda función sobrescribe a la anterior
miBoton.onclick = function () {
  console.log("Segunda función");
};
```

## addEventListener() (El método recomendado)

Como bien mencionaste, este es el método moderno y estándar. Permite registrar múltiples funciones de controlador para un solo evento en un solo elemento.

Ventajas:

- Flexibilidad: Puedes añadir varios controladores de eventos al mismo elemento sin sobrescribir los anteriores.

- Control: Ofrece un tercer parámetro opcional que te da un control avanzado sobre la propagación del evento (true para la fase de captura).

- Removible: Puedes eliminar un controlador de eventos con removeEventListener().

Desventajas:

- La sintaxis es un poco más larga que la de los otros métodos.

```javascript
const miBoton = document.getElementById("miBoton");

// Puedes añadir múltiples funciones al mismo evento
miBoton.addEventListener("click", function () {
  console.log("Primera función");
});

miBoton.addEventListener("click", function () {
  console.log("Segunda función");
});
```

> [!NOTE]
> Para cualquier proyecto serio, addEventListener() es la opción clara. Te da la flexibilidad, el control y la separación de preocupaciones necesaria para construir aplicaciones web robustas y fáciles de mantener.
