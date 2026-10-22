<h1 align="center">Importación y Namespace</h1>

<h2>📑 Contenido</h2>

- [@import](#import)
- [@namespace](#namespace)

## @import

La regla `@import` se utiliza para importar estilos desde otro archivo CSS o una URL externa y combinarlos con el archivo CSS actual. Permite dividir el código CSS en varios archivos y facilita la organización y mantenimiento del código.

**Sintaxis**

```css
/* CSS */

@import url("ruta/al/otro-archivo.css");
```

**Ejemplos**

```css
/* CSS */

@import url("archivo1.css");
@import url("print.css") print;
/* 
Será importado únicamente cuando se esté imprimiendo la página. La condición print se utiliza para especificar que estos estilos deben aplicarse únicamente cuando el documento se está imprimiendo
*/
@import url("common-styles.css") screen;
/* 
Los estilos definidos en common-styles.css se aplicarán cuando la página se visualice en pantalla. La condición screen especifica que estos estilos deben aplicarse solo cuando se está visualizando en un dispositivo de pantalla, como un monitor o dispositivo móvil.
*/
@import url("landscape.css") screen and (orientation: landscape);
/* 
Esto indica que el archivo landscape.css será importado solo cuando se cumplan dos condiciones:

Medio (screen): Se aplica solo si el medio de presentación es una pantalla.

Orientación (orientation: landscape): Se aplica solo cuando la orientación de la pantalla es horizontal (paisaje).
*/
```

> [!WARNING]
>
> Tener en cuenta que las reglas @import son generales y deben estar al principio del archivo CSS, antes de cualquier otra regla, para garantizar que los estilos importados se apliquen correctamente.

## @namespace

La regla `@namespace` en CSS se utiliza para declarar un espacio de nombres (namespace). Se utiliza para asociar un espacio de nombres con un bloque de reglas de estilo CSS. Esto es especialmente útil cuando se trabaja con XML y XHTML, donde los elementos pueden estar definidos en diferentes espacios de nombres.

Esta regla es más comúnmente utilizada en el contexto de XML y XHTML, donde se pueden definir diferentes espacios de nombres para elementos específicos.

**Sintaxis**

```css
/* CSS */

@namespace prefix url(namespaceURI);

/* 
prefix: Es el identificador que se utiliza como prefijo para los elementos del espacio de nombres.

url(namespaceURI): Es la URL que especifica la ubicación del espacio de nombres.
*/
```

**Ejemplo**

```css
/* CSS */
@namespace svg url(http://www.w3.org/2000/svg);

svg|circle {
  fill: red;
}
```

> [!NOTE]
>
> En la práctica, en el desarrollo web estándar, no es necesario utilizar @namespace a menos que estés trabajando específicamente con XML o XHTML. La mayoría de los desarrolladores web que trabajan con HTML no necesitan preocuparse por esta regla.
