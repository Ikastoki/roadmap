<h1 align="center">Formulario</h1>

<h2>📑 Contenido</h2>

- [Formulario](#formulario)

## Formulario

Los formularios HTML son una herramienta fundamental para la recolección de datos de los usuarios en las páginas web. Para insertar formularios en HTML se utiliza la etiqueta `<form>`. La etiqueta form contiene/agrupa todos los elementos necesarios para la creación de un formulario. Campos para insertar datos, controles de opciones, selectores... Para que un formulario funcione de manera correcta es necesario que tenga un botón para enviar los datos, un atributo method que le indique la forma de envió y un action para remitir los datos.

**Atributos(Más comunes):**

- **action:** contiene el nombre del agente que procesará los datos remitidos al servidor (por ejemplo, un script de PHP)
- **method:** define la manera de enviar los datos al servidor. Los valores posibles son:
  - **get:** los valores enviados se añaden a la dirección indicada en el atributo action.
  - **post:** los valores se envían de forma separada.
- Si el atributo method no está establecido, el formulario se comporta como si el valor fuera **get**.


