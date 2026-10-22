<h1 align="center">Acciones Predeterminadas</h1>

<h2>📑 Contenido</h2>

- [¿Qué son las Acciones Predeterminadas?](#qué-son-las-acciones-predeterminadas)
- [Cómo Prevenir las Acciones Predeterminadas](#cómo-prevenir-las-acciones-predeterminadas)
  - [Prevenir la navegación en un enlace](#prevenir-la-navegación-en-un-enlace)
  - [Prevenir el envío de un formulario](#prevenir-el-envío-de-un-formulario)

## ¿Qué son las Acciones Predeterminadas?

Muchos eventos tienen un comportamiento por defecto asociado a ellos que el navegador realiza de forma automática. Por ejemplo:

- Un clic en un enlace (`<a>`) te redirige a la URL especificada en su atributo `href`.

- Un clic en un botón de tipo `submit` en un formulario (`<form>`) envía los datos del formulario y recarga la página.

- Presionar la barra espaciadora o una tecla de flecha en una página activa el `scroll`.

- El clic derecho del ratón abre el menú contextual del navegador.

A menudo, al manejar eventos con JavaScript, queremos evitar este comportamiento predeterminado para poder realizar nuestra propia lógica.

## Cómo Prevenir las Acciones Predeterminadas

La forma más común y recomendada de evitar estas acciones es usando el método `preventDefault()` del objeto de evento. Este método se invoca dentro de la función del controlador de eventos.

### Prevenir la navegación en un enlace

Imagina que tienes un enlace, pero en lugar de que te redirija, quieres que ejecute una función de JavaScript.

```html
<a href="https://ejemplo.com" id="enlace">Ver más</a>
```

```javascript
document.getElementById("enlace").addEventListener("click", function (e) {
  // e.preventDefault() evita que el navegador navegue a ejemplo.com
  e.preventDefault();

  console.log("Se ejecutó mi función en lugar de la redirección.");
  // Aquí puedes poner tu lógica, por ejemplo, mostrar un modal
});
```

En este caso, al hacer clic en el enlace, la redirección se detiene, y solo se ejecuta el código que has escrito en la función.

### Prevenir el envío de un formulario

Cuando envías un formulario con un botón de `type="submit"`, la página se recarga. Si quieres validar los datos con JavaScript antes de enviarlos (o enviarlos con AJAX sin recargar la página), debes prevenir este comportamiento.

```html
<form id="formulario">
  <input type="text" name="nombre" />
  <button type="submit">Enviar</button>
</form>
```

```javascript
document.getElementById("formulario").addEventListener("submit", function (e) {
  // e.preventDefault() detiene el envío del formulario y la recarga de la página
  e.preventDefault();

  // Aquí puedes ejecutar tu lógica de validación
  console.log("Validando el formulario...");
  // Si la validación es correcta, podrías enviar los datos con `fetch()`
});
```

Al usar `preventDefault()`, tienes el control total para decidir qué sucede después de que se produce un evento.

> [!NOTE]
> Es importante destacar que preventDefault() solo funciona para las acciones que son "cancelables". La mayoría de las acciones predeterminadas lo son, pero hay excepciones. Puedes verificar si un evento es cancelable con la propiedad e.cancelable.
