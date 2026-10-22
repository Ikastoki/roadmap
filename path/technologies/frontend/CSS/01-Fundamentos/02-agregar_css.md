<h1 align="center">Agregar CSS</h1>

<h2>📑 Contenido</h2>

- [CSS Interno](#css-interno)
- [CSS Externo](#css-externo)
- [CSS en linea(Inline)](#css-en-lineainline)
- [Importar](#importar)
  - [@import \& link](#import--link)

## CSS Interno

El CSS interno se agrega en la sección del [`<head>`](../../01-HTML/02-HEAD/01-head.md) de un documento HTML.

Al añadir el CSS en la misma página que el HTML no necesita cargar varios archivos. Pero aumenta el tamaño del HTML y puede ralentizar las cargas.

**Ejemplo:**

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
    <!-- Añadir CSS interno -->
    <style>
      h1 {
        color: red;
      }
    </style>
  </head>
  <body>
    <h1>Título de la Página</h1>
  </body>
</html>
```

**Resultado**

![Resultado CSS Interno](./img/AgregarCSS.png)

## CSS Externo

El CSS externo es un documento que se tiene que enlazar en la sección del `<head>` de un documento HTML.
Este método es el mas usado, permite una mayor organización.

Sintaxis: <br>
`<link rel="stylesheet" href="ruta/del/archivo/CSS">`

Arquitectura de directorios:

- index.html
- CSS
  - style.css

**Ejemplo:**

```html
<!-- Archivo HTML -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
    <link rel="stylesheet" href="CSS/style.css" />
  </head>
  <body>
    <h1>Título de la Página</h1>
  </body>
</html>
```

```css
/* Archivo CSS */
h1 {
  color: red;
}
```

**El Resultado es el mismo que con el ejemplo Interno.**

![Resultado CSS Interno](./img/AgregarCSS.png)

## CSS en linea(Inline)

El CSS en linea se aplica directamente en el elemento HTML mediante el uso del atributo `style=""`

Sintaxis: <br>
` <h1 style="color: red">Título de la Página</h1>`

## Importar

Mediante el uso de **@import** se puede incluir una hoja de estilos externa en un archivo CSS.

Sintaxis: <br>
`@import url("ruta/al/archivo.css");`

Sintaxis: <br>
`@import "ruta/al/archivo";`

### @import & link

La diferencia entre @import y link es que @import es el mecanismo de CSS para incluir una hoja de estilo externa, mientras que link es el mecanismo de HTML.

Algunas diferencias son:

- link se carga en paralelo con el resto de la página, mientras que @import se carga de forma secuencial, lo que puede retrasar la carga de la página.
- link permite especificar atributos como rel, type o media, mientras que @import solo permite especificar el media query.
- link es compatible con todos los navegadores, mientras que @import no funciona en algunos navegadores antiguos.


