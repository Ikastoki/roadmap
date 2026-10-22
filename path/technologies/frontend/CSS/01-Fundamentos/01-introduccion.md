<h1 align="center">Introducción CSS</h1>

<h2>📑 Contenido</h2>

- [¿Qué es CSS?](#qué-es-css)
- [¿Para qué sirve CSS?](#para-qué-sirve-css)
- [Reglas CSS](#reglas-css)
  - [Propiedades](#propiedades)

## ¿Qué es CSS?

CSS (Cascading Style Sheets) es un lenguaje de hojas de estilo utilizado para definir el aspecto y formato de un documento HTML (o XML). Permite a los desarrolladores web controlar el diseño, la presentación y el estilo de múltiples páginas web al mismo tiempo, asegurando una consistencia visual en todo el sitio.

CSS se utiliza ampliamente en el desarrollo web moderno y es fundamental para crear sitios web atractivos y funcionales. Permite la separación de la estructura del documento HTML y su presentación visual, lo que facilita el mantenimiento y la actualización de los sitios web a lo largo del tiempo.

## ¿Para qué sirve CSS?

- **Estilo visual:** CSS se utiliza principalmente para definir el aspecto visual de los elementos HTML en una página web. Esto incluye propiedades como color de texto, tamaño de fuente, tipo de fuente, espaciado, márgenes, bordes, fondos, etc. Permite controlar el diseño y la presentación de la página.
- **Consistencia:** Permite mantener una apariencia consistente en todo un sitio web. Con CSS, puedes definir estilos comunes y aplicarlos a múltiples elementos HTML, lo que garantiza coherencia en la apariencia de todas las páginas.
- **Adaptabilidad:** CSS facilita la creación de diseños responsivos y adaptables a diferentes dispositivos y tamaños de pantalla. Esto se logra mediante la definición de estilos que se ajustan automáticamente según el tamaño de la pantalla del dispositivo, lo que mejora la experiencia del usuario tanto en ordenadores de escritorio como en dispositivos móviles.
- **Separación de preocupaciones:** CSS permite separar la estructura del contenido HTML de su presentación visual. Esto promueve un mejor mantenimiento del código, ya que los cambios en el estilo se pueden realizar sin afectar la estructura del documento HTML subyacente.
- **Accesibilidad:** CSS proporciona herramientas para mejorar la accesibilidad web al permitir la creación de diseños y estilos que se adaptan a las necesidades de los usuarios con discapacidades visuales o motoras.

## Reglas CSS

Una regla de CSS es el conjunto de propiedades que se aplica a un elemento.

Sintaxis: `body { color: tomato}`

- **Regla de CSS:** body { color: tomato}
- **Selector:** body
- **{:** Llave de apertura
- **Declaración:** { color: tomato}
- **Propiedad:** color
- **Valor:** tomato
- **}:** Llave de cierre

> [!NOTE]
>
> Las propiedades pueden tener más de un valor.
> <br>
> Algunas propiedades pueden hacer uso de una version reducida(abreviada).

```css
/* Version larga */
border-width: 1px;
border-style: solid;
border-color: black;

/* Version reducida */
border: 1px solid black;
```

### Propiedades

CSS ofrece una amplia variedad de propiedades que permiten controlar el aspecto visual y el diseño de los elementos en una página web.

**Propiedades más comunes:**

- Color y Fondo:
  - **color:** Define el color del texto.
  - **background-color:** Establece el color de fondo de un elemento.
  - **background-image:** Especifica una imagen de fondo.
- Tipografía:
  - **font-family:** Define la familia de fuentes para el texto.
  - **font-size:** Establece el tamaño de la fuente.
  - **font-weight:** Define el grosor de la fuente (p. ej., normal, bold).
  - **font-style:** Especifica el estilo de la fuente (p. ej., normal, italic).
- Espaciado y Dimensiones:
  - **margin:** Define el margen alrededor de un elemento.
  - **padding:** Establece el espacio interior alrededor del contenido de un elemento.
  - **width, height:**Define el ancho y la altura de un elemento.
- Bordes:
  - **border:** Establece el borde de un elemento (ancho, estilo y color).
  - **border-radius:** Define la curvatura de los bordes de un elemento.
- Alineación y Diseño:
  - **text-align:** Especifica la alineación horizontal del texto.
  - **vertical-align:** Define la alineación vertical de un elemento en relación con su contenedor.
  - **display:** Controla el comportamiento de visualización de un elemento (p. ej., block, inline, inline-block).
- Posicionamiento:
  - **position:** Define el tipo de posicionamiento de un elemento (p. ej., static, relative, absolute, fixed).
  - **top, bottom, left, right:** Establece la posición de un elemento en relación con su contenedor posicionado.

> [!NOTE]
>
> Breve introducción, hay muchísimas propiedades mejor verlas cada una en su bloque.


