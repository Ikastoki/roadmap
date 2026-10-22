<h1 align="center">Gestionar y manipular datos</h1>

<h2>📑 Contenido</h2>

- [Gestionar y manipular datos](#gestionar-y-manipular-datos)
- [Acceder a los elementos del formulario](#acceder-a-los-elementos-del-formulario)
- [Obtener los valores de los campos](#obtener-los-valores-de-los-campos)
- [Establecer y modificar valores](#establecer-y-modificar-valores)
- [Obtener todos los datos de un formulario](#obtener-todos-los-datos-de-un-formulario)

## Gestionar y manipular datos

La gestión y manipulación de datos en formularios es una de las tareas más comunes e importantes en el desarrollo web con JavaScript. Se trata de cómo capturar, procesar y trabajar con la información que el usuario introduce en los campos de un formulario.

## Acceder a los elementos del formulario

Antes de poder manipular los datos, necesitas obtener una referencia a los elementos del formulario en el DOM. Puedes hacerlo de varias maneras:

- Por `id`: `document.getElementById('miCampo')`

- Por `name`: `document.querySelector('input[name="miCampo"]')` o `document.forms['nombreFormulario'].miCampo`

- Por `clase`: `document.querySelector('.miClase')`

## Obtener los valores de los campos

Una vez que tienes una referencia a un elemento, puedes acceder a su valor usando la propiedad .value.

```html
<input type="text" id="nombreUsuario" value="Juan" />
```

```javascript
const inputNombre = document.getElementById("nombreUsuario");
console.log(inputNombre.value); // Salida: "Juan"
```

El `.value` funciona para la mayoría de los tipos de `input` (`text`, `email`, `password`, `number`), `textarea` y `select`.

**Casos especiales:**

- `checkbox`: En lugar de `.value`, la propiedad más útil es `.checked`, que devuelve true si la casilla está marcada y `false` si no lo está.

- `select`: Para obtener el valor de la opción seleccionada, accedes a `.value` del elemento `select`.

```html
<select id="opciones">
  <option value="opcion1">Opción 1</option>
  <option value="opcion2">Opción 2</option>
</select>
```

```javascript
const selectElement = document.getElementById("opciones");
console.log(selectElement.value);
```

## Establecer y modificar valores

También puedes cambiar el valor de un campo directamente desde JavaScript asignando un nuevo valor a la propiedad .value.

```javascript
const inputNombre = document.getElementById("nombreUsuario");
inputNombre.value = "Ana"; // El campo de texto ahora muestra "Ana"
```

Esto es útil para:

- Pre-llenar formularios.

- Limpiar formularios después de enviarlos.

- Modificar dinámicamente el valor de un campo.

## Obtener todos los datos de un formulario

Para formularios complejos, no es eficiente obtener cada valor de forma individual. Puedes usar la clase `FormData` para capturar todos los datos de un formulario de forma sencilla.

```html
<form id="miFormulario">
  <input type="text" name="nombre" />
  <input type="email" name="correo" />
</form>
```

```javascript
const formulario = document.getElementById("miFormulario");

formulario.addEventListener("submit", function (e) {
  e.preventDefault(); // Evita el envío del formulario por defecto

  const datos = new FormData(formulario);

  // Puedes iterar sobre los datos
  for (const [key, value] of datos.entries()) {
    console.log(`${key}: ${value}`);
  }

  // O convertirlos a un objeto simple
  const objetoDatos = Object.fromEntries(datos);
  console.log(objetoDatos); // { nombre: "...", correo: "..." }
});
```

La clase `FormData` te da una forma estandarizada de trabajar con los datos del formulario, especialmente útil para enviarlos a una API.
