<h1 align="center">Interactividad</h1>

<h2>📑 Contenido</h2>

- [Interactividad](#interactividad)
- [Mostrar u Ocultar Campos Dinámicamente](#mostrar-u-ocultar-campos-dinámicamente)
- [Autocompletar y Sugerir Datos](#autocompletar-y-sugerir-datos)
- [Formularios Dinámicos](#formularios-dinámicos)

## Interactividad

La interactividad en los formularios es lo que eleva la experiencia del usuario de un simple proceso de llenado a una interacción fluida y guiada.

## Mostrar u Ocultar Campos Dinámicamente

Esta es una técnica común para mantener los formularios simples y relevantes. La idea es mostrar campos solo cuando son necesarios, basándose en las selecciones del usuario. Esto evita abrumar al usuario con demasiada información.

Para lograrlo, puedes usar `addEventListener` para escuchar eventos como `change` o `click` en un campo de selección. Luego, accedes a los elementos que quieres ocultar o mostrar y manipulas su estilo con CSS (por ejemplo, usando `display: none` o `display: block`) o su clase (`classList.toggle()`).

Ejemplo: Mostrar un campo de "información adicional" si el usuario marca una casilla.

```html
<label
  ><input type="checkbox" id="tiene-info" /> ¿Desea añadir más
  información?</label
>
<div id="campo-adicional" style="display: none;">
  <label for="texto-info">Información adicional:</label>
  <textarea id="texto-info"></textarea>
</div>
```

```javascript
const checkbox = document.getElementById("tiene-info");
const campoAdicional = document.getElementById("campo-adicional");

checkbox.addEventListener("change", function () {
  if (this.checked) {
    campoAdicional.style.display = "block";
  } else {
    campoAdicional.style.display = "none";
  }
});
```

## Autocompletar y Sugerir Datos

El autocompletado reduce la cantidad de escritura del usuario y minimiza los errores. Se puede implementar de varias maneras:

- Con `datalist`: Este elemento de HTML5 te permite ofrecer una lista de sugerencias predefinidas para un campo `input`.

- Llamadas a API: Puedes escuchar los eventos `input` o `keyup` para, por ejemplo, realizar una búsqueda en una base de datos o en una API externa, y luego mostrar las sugerencias en un menú desplegable. Esta técnica requiere `debouncing` para evitar un exceso de peticiones.

## Formularios Dinámicos

Los formularios dinámicos son aquellos donde los campos se crean o se eliminan en tiempo real, basándose en la interacción del usuario. Esto es muy útil para casos como añadir múltiples artículos a un pedido o subir múltiples archivos.

Para crear formularios dinámicos:

1. Escucha un evento (por ejemplo, el clic en un botón `+`).

1. Crea nuevos elementos del DOM (`document.createElement()`).

1. Asigna sus atributos y contenido (`.setAttribute()`, `.innerText`).

1. Inserta los nuevos elementos en el formulario (`.appendChild()`).

```html
<form>
  <div id="emails-container">
    <input type="email" name="email" />
  </div>
  <button type="button" id="add-email">Añadir otro email</button>
</form>
```

```javascript
const botonAdd = document.getElementById("add-email");
const contenedorEmails = document.getElementById("emails-container");

botonAdd.addEventListener("click", function () {
  const nuevoInput = document.createElement("input");
  nuevoInput.type = "email";
  nuevoInput.name = "email";
  contenedorEmails.appendChild(nuevoInput);
});
```
