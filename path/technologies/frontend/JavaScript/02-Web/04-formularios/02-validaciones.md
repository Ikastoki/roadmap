<h1 align="center">Validar formulario</h1>

<h2>📑 Contenido</h2>

- [Validar formulario](#validar-formulario)
- [Validación en el Lado del Cliente (JavaScript)](#validación-en-el-lado-del-cliente-javascript)
  - [Propiedades de validación de HTML5](#propiedades-de-validación-de-html5)
  - [Validación con JavaScript](#validación-con-javascript)
- [Validación en el Lado del Servidor](#validación-en-el-lado-del-servidor)
- [Buenas Prácticas de Validación](#buenas-prácticas-de-validación)

## Validar formulario

La validación de formularios es un paso crucial para garantizar que los datos que un usuario introduce son correctos y seguros antes de ser procesados o enviados. Se puede realizar en dos lugares principales: en el lado del cliente (con JavaScript) y en el lado del servidor(la importante).

## Validación en el Lado del Cliente (JavaScript)

La validación del lado del cliente es el primer filtro. Proporciona una retroalimentación inmediata al usuario, mejorando la experiencia de usuario. Su objetivo es evitar envíos innecesarios y guiar al usuario.

### Propiedades de validación de HTML5

Antes de recurrir a JavaScript, puedes usar atributos de validación de HTML5, que el navegador maneja por defecto.

- `required`: El campo no puede estar vacío.

- `type="email"`: Valida que el texto tenga el formato de un correo electrónico.

- `minlength / maxlength`: Define la longitud mínima y máxima del texto.

- `pattern`: Te permite usar una expresión regular para una validación más compleja.

```html
<input
  type="email"
  id="correo"
  required
  pattern="[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$"
/>
```

### Validación con JavaScript

Puedes complementar la validación de HTML5 o crear una lógica de validación más sofisticada con JavaScript. El método más común es escuchar el evento submit del formulario y usar preventDefault() para detener el envío si los datos no son válidos.

```javascript
const formulario = document.getElementById("miFormulario");

formulario.addEventListener("submit", function (e) {
  const correo = document.getElementById("correo");

  if (correo.value === "") {
    e.preventDefault(); // Detiene el envío
    alert("El correo es obligatorio.");
    correo.focus();
  }
  // Puedes añadir más validaciones aquí...
});
```

## Validación en el Lado del Servidor

La validación del lado del cliente es útil, pero no es suficiente. Un usuario malicioso puede desactivar JavaScript y enviar datos no válidos. Por lo tanto, la validación del lado del servidor es obligatoria para la seguridad de tu aplicación.

- El servidor valida los datos después de que el formulario se ha enviado, antes de que se procesen o guarden en una base de datos.

- Si los datos son inválidos, el servidor debería enviar una respuesta con un código de error y un mensaje de error que luego JavaScript puede mostrar al usuario.

## Buenas Prácticas de Validación

- Valida siempre en el servidor: Considera la validación del lado del cliente como una conveniencia para el usuario, no como una medida de seguridad.

- Proporciona una buena retroalimentación: Los mensajes de error deben ser claros, concisos y útiles. Diles a los usuarios exactamente qué deben corregir.

- Usa validación en tiempo real: Puedes validar campos a medida que el usuario escribe (usando los eventos `input` o `change`) para dar una retroalimentación inmediata.

- Destaca los campos erróneos: Usa CSS para cambiar el borde de los campos que no son válidos, haciendo que sean fáciles de identificar.
