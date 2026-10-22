<h1 align="center">API Form</h1>

<h2>📑 Contenido</h2>

- [API Form](#api-form)
- [form.elements](#formelements)
- [Propiedades de Validación](#propiedades-de-validación)
- [Métodos de la API Form](#métodos-de-la-api-form)
- [¿Por qué usar la API Form?](#por-qué-usar-la-api-form)

## API Form

La API Form es un conjunto de propiedades y métodos en JavaScript que te permiten interactuar con la validación de formularios de HTML5, dándote un control total sobre ella. En lugar de tener que reinventar la rueda con tu propia lógica de validación, puedes usar estas herramientas del navegador para una solución más robusta y estándar.

## form.elements

La propiedad `form.elements` te proporciona una colección de todos los elementos de un formulario, facilitando el acceso a ellos. Es más útil que usar `document.querySelectorAll()` para la mayoría de los casos.

```html
<form id="miFormulario">
  <input type="text" name="nombre" />
  <input type="email" name="correo" />
</form>
```

```javascript
const formulario = document.getElementById("miFormulario");
console.log(formulario.elements.nombre.value);
console.log(formulario.elements.correo.value);

// También puedes iterar sobre ellos
for (const elemento of formulario.elements) {
  console.log(elemento.name, elemento.value);
}
```

## Propiedades de Validación

Estas propiedades te permiten verificar el estado de validación de un campo en cualquier momento.

- `element.checkValidity()`: Un método que verifica si el valor de un campo es válido según las reglas de HTML5 (`required`, `pattern`, etc.). Devuelve `true` o `false`.

- `element.validity`: Es un objeto que contiene el estado de validación de un campo con propiedades booleanas como `valueMissing`, `typeMismatch`, `patternMismatch`, etc. Es útil para saber por qué un campo no es válido.

- `element.validationMessage`: Una propiedad que devuelve el mensaje de error de validación predeterminado del navegador, útil para mostrarlo al usuario.

```html
<input type="email" id="correo" required />
```

```javascript
const correoInput = document.getElementById("correo");

// Verificar la validez al cambiar el valor
correoInput.addEventListener("input", function () {
  if (correoInput.checkValidity()) {
    console.log("El correo es válido.");
  } else {
    // Si no es válido, podemos ver por qué
    console.log(correoInput.validationMessage);
    // Y el estado de validación
    console.log(correoInput.validity.typeMismatch); // true si el formato es incorrecto
  }
});
```

Con `checkValidity()`, no tienes que escribir tu propia lógica para cada tipo de validación.

## Métodos de la API Form

- `element.setCustomValidity(message)`: Un método muy potente que te permite establecer un mensaje de error de validación personalizado. Si la cadena está vacía, el campo se considera válido. Si tiene un mensaje, se considera inválido.

- `form.reportValidity()`: Fuerza al navegador a mostrar los mensajes de error de validación para todos los campos inválidos de un formulario, sin necesidad de que el usuario haga clic en `submit`.

```javascript
const passwordInput = document.getElementById("contrasena");
const confirmPasswordInput = document.getElementById("confirmar-contrasena");

confirmPasswordInput.addEventListener("input", function () {
  if (passwordInput.value !== confirmPasswordInput.value) {
    // Si las contraseñas no coinciden, establecemos un mensaje de error personalizado
    confirmPasswordInput.setCustomValidity("Las contraseñas no coinciden.");
  } else {
    // Si coinciden, vaciamos el mensaje para que el campo sea válido
    confirmPasswordInput.setCustomValidity("");
  }
});
```

En este caso, la API del formulario maneja la visualización del mensaje de error, y solo tienes que preocuparte por la lógica de la validación.

## ¿Por qué usar la API Form?

- Consistencia: Utiliza las validaciones nativas del navegador, lo que asegura que tu formulario funcione de manera predecible y coherente.

- Simplicidad: Te ahorra tiempo y código al no tener que recrear validaciones comunes (`required`, `email`, etc.).

- Accesibilidad: Los mensajes de validación y el foco de error del navegador ya están optimizados para la accesibilidad.
