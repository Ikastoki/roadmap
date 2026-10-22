<h1 align='center'>Agregar o enlazar JavaScript</h1>

<h2>📑 Contenido</h2>

- [Maneras de agregar o enlazar JavaScript a un documento HTML](#maneras-de-agregar-o-enlazar-javascript-a-un-documento-html)
- [JavaScript embebido](#javascript-embebido)
- [Ubicación del script](#ubicación-del-script)
  - [Script en el `<head>`](#script-en-el-head)
  - [Script al final del `<body>`](#script-al-final-del-body)
  - [Tabla comparativa](#tabla-comparativa)
- [JavaScript en archivos externos](#javascript-en-archivos-externos)
- [Controlar la carga del script](#controlar-la-carga-del-script)
  - [Atributo defer](#atributo-defer)
  - [Atributo async](#atributo-async)

## Maneras de agregar o enlazar JavaScript a un documento HTML

JavaScript es el lenguaje que permite dotar de interactividad a una página web. Dependiendo de las necesidades de tu proyecto, puedes incluir el código JavaScript directamente en la página HTML o cargarlo desde un archivo externo. En este capítulo, exploraremos las maneras más comunes de hacerlo: scripts embebidos en el archivo HTML y archivos JavaScript externos. Los módulos de JavaScript, más avanzados, se tratarán en una sección dedicada más adelante.

## JavaScript embebido

Una de las formas más sencillas de agregar JavaScript a un documento HTML es incluir el código directamente dentro de las etiquetas `<script>`. Este enfoque es útil para páginas pequeñas o cuando el código no necesita ser reutilizado en otras páginas. Sin embargo, en aplicaciones más grandes, este método puede complicar el mantenimiento, ya que se mezcla el HTML con JavaScript.

Ejemplo:

```js
<script>
  function saludar() {
    alert("¡Hola desde JavaScript embebido!");
  }
</script>
```

> [!NOTE]
> Si la etiqueta `<script>` contiene el atributo `src`, el contenido dentro de la etiqueta no funcionará, ya que estará enlazado a un archivo externo.

## Ubicación del script

La ubicación de la etiqueta `<script>` dentro del documento HTML afecta directamente cómo y cuándo se carga y ejecuta el código JavaScript en relación con el contenido HTML. Las dos ubicaciones más comunes son dentro del `<head>` o al final del `<body>`. Cada una tiene implicaciones importantes para el rendimiento y el comportamiento de la página.

### Script en el `<head>`

Si colocas un script dentro del elemento `<head>`, el navegador descargará y ejecutará el código JavaScript antes de cargar el contenido del cuerpo del documento. Esto puede tener implicaciones para la experiencia del usuario y el rendimiento de la página.

- **Bloqueo del renderizado:** El navegador detiene la carga del contenido de la página hasta que el script ha sido completamente descargado y ejecutado, lo que puede hacer que la página se vea en blanco durante unos momentos.
- **El DOM aún no está disponible:** El código JavaScript se ejecuta antes de que el contenido del `<body>` esté completamente cargado en el DOM (Modelo de Objetos del Documento). Si el script intenta interactuar con elementos HTML que aún no existen, se generarán errores.

> [!TIP]
> Colocar scripts en el `<head>` es útil si necesitas ejecutar código antes de que cualquier contenido de la página esté disponible, como configuraciones globales o bibliotecas que no interactúan con el DOM. Sin embargo, esto debe evitarse cuando el script depende de los elementos del DOM.

### Script al final del `<body>`

Colocar el script al final del documento, justo antes del cierre del elemento `<body>`, permite que todo el contenido de la página se cargue primero, y luego el código JavaScript se ejecuta.

- **Mejor rendimiento:** El contenido de la página se muestra primero, mejorando la percepción de velocidad y la experiencia del usuario. El JavaScript se ejecuta una vez que todo el DOM ha sido completamente construido.
- **El DOM está disponible:** El script tiene acceso a todos los elementos HTML, ya que se ejecuta después de que el DOM ha sido completamente cargado.

> [!TIP]
> Para la mayoría de los casos, es una buena práctica colocar los scripts al final del `<body>` cuando interactúan con elementos del DOM, lo que además mejora el rendimiento de la página.

### Tabla comparativa

| En el Head                                                                    | En el Body                                                                         |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| El script se ejecuta antes de cargar el DOM.                                  | El script se ejecuta después de cargar todo el DOM.                                |
| Puede bloquear el renderizado del contenido.                                  | Permite que el contenido se renderice antes de ejecutar el script.                 |
| Riesgo de errores si intentas manipular elementos del DOM que aún no existen. | Acceso completo al DOM, ya que todo está cargado cuando se ejecuta el script.      |
| Recomendado para bibliotecas o configuraciones globales.                      | Recomendado para scripts que interactúan con el DOM o para mejorar el rendimiento. |

## JavaScript en archivos externos

En proyectos más grandes o cuando el código necesita ser reutilizado en múltiples páginas, es recomendable separar el JavaScript en archivos .js externos. Para enlazarlos a tu documento HTML, utilizas la etiqueta `<script>` con el atributo src.

Este enfoque facilita la organización y mantenimiento del código, pero ten en cuenta que múltiples archivos externos pueden aumentar el tiempo de carga. Sin embargo, técnicas como la minificación y el caché pueden mitigar este efecto.

Archivo HTML:

```html
<button onclick="saludar()">Saludar</button>
<script src="scripts.js"></script>
```

Archivo JS:

```js
function saludar() {
  alert("¡Hola desde un archivo externo!");
}
```

> [!WARNING]
> La ubicación del script en el documento HTML sigue siendo importante, como se explicó en la sección anterior.

## Controlar la carga del script

Cuando enlazas un archivo JavaScript externo, hay dos atributos clave que puedes usar para controlar cómo y cuándo se carga y ejecuta el script: `defer` y `async`.

### Atributo defer

El atributo defer asegura que el script se cargue en paralelo con el documento HTML, pero no se ejecutará hasta que todo el DOM esté completamente cargado. Esto significa que el script no bloqueará la carga del HTML y se ejecutará en el momento adecuado, cuando el DOM esté listo para ser manipulado.

Ejemplo:

```html
<script src="scripts.js" defer></script>
```

### Atributo async

El atributo async también permite que el script se cargue en paralelo, pero lo ejecuta tan pronto como está disponible, lo que puede ocurrir antes de que el DOM esté completamente cargado. Esto es útil para scripts que no dependen del DOM, ya que mejora el rendimiento. Sin embargo, no garantiza el orden de ejecución si hay varios scripts.

Ejemplo:

```html
<script src="scripts.js" async></script>
```

> [!TIP]
> Usa defer para scripts que dependen del DOM y async para scripts independientes o para optimizar el rendimiento de scripts que no necesitan interactuar con el contenido de la página.

Con estas opciones, tienes flexibilidad para incluir y controlar la ejecución de JavaScript en tu página web, optimizando tanto el rendimiento como la organización del código.