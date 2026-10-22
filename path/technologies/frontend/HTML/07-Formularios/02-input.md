<h1 align="center">Inputs</h1>

<h2>📑 Contenido</h2>

- [Inputs](#inputs)
  - [Atributos](#atributos)
  - [Datalist](#datalist)

## Inputs

Los elementos input sirven para crear campos de texto básicos de una sola línea.

Sintaxis: <br>
`<input type="tipo-del-campo">`

### Atributos

- **type:** Asigna un tipo especifico al campo input.

  - **Submit:** Botón para enviar el formulario.
  - **Reset:** Restablece los valores iniciales del formulario.
  - **Text:** Cajas de texto de una sola línea.
  - **Password:** Para contraseñas.
  - **File:** Para archivos.
  - **Image:** Envío gráficos.
  - **Hidden:** Incluir datos que los usuarios no pueden ver ni modificar.
  - **Button:** Botón.
  - **Number:** Solo números.
  - **Search:** Consultas de búsqueda.
  - **Tel:** Número de teléfono.
  - **Url:** Para insertar y editar URL.
  - **Email:** Para insertar correos.
  - **Date:** Especial fechas.
  - **Time:** Horas-Minutos (segundos).
  - **Radio:** Seleccionar una opción.
  - **Checkbox:** Seleccionar multiples opciones.
  - **Search:** Ingresar consultas de búsqueda.
  - **URL:** Ingresar y editar una URL.
  - **Color:** Permite a los usuarios especificar un color mediante un control que simula un cuadro cromático.
  - **Range:** Establecer un Rango.
  - **Month:** Mes.
  - **Week:** Semana.
  - **Datetime-local:** Para ingresar tiempo y fecha (hora, minuto, segundo y fracción de segundo) basado en la zona horaria UTC.

- **name:** El atributo name se usa para identificar el campo. Sólo se envían los campos que lo tienen.
- **value:** El atributo value permite establecer el valor inicial de un campo.
- **required:** Indica que el campo es obligatorio. No se enviaran los datos si no se rellena.
- **size:** Permite establecer la longitud de los controles de texto.
- **maxlength y minlength:** Permiten establecer la longitud máxima y mínima, respectivamente.
- **autofocus:** Indica el control que debe tener el foco al cargarse el formulario.

---

### Datalist

La etiqueta `<datalist>` se utiliza en combinación con la etiqueta `<input> `para crear listas desplegables o sugerencias de entrada de datos para los usuarios.

Proporciona una lista de opciones que se asocian con un campo de entrada y permite que los usuarios elijan valores de esa lista o ingresen valores personalizados.

```html
<label for="frutas">Seleccione una fruta:</label>
<input list="frutas" name="fruta" id="fruta" />
<datalist id="frutas">
  <option value="Manzana"></option>
  <option value="Banana"></option>
  <option value="Cereza"></option>
  <option value="Uva"></option>
  <option value="Naranja"></option>
</datalist>
```

> [!NOTE]
>
> La etiqueta `<datalist>` proporciona una experiencia de usuario más amigable al ofrecer opciones predefinidas y ayudar a evitar errores tipográficos.


