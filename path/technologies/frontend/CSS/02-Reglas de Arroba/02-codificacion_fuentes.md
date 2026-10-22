<h1 align="center">Codificación y Fuentes</h1>

<h2>📑 Contenido</h2>

- [@charset](#charset)
- [@color-profile](#color-profile)
- [@font-face](#font-face)
- [@font-feature-values](#font-feature-values)

## @charset

La regla `@charset` se utiliza para especificar el conjunto de caracteres (charset) del documento CSS. Esta regla debe aparecer al principio del archivo CSS y antes de cualquier otra regla o declaración. Su propósito es indicar al navegador o agente de usuario cuál es el conjunto de caracteres que se está utilizando en el archivo CSS.

**Ejemplo:**

```css
/* CSS */

/* Regla @charset */
@charset "UTF-8";

/* Resto de las reglas CSS van después de @charset */
body {
  font-family: Arial, sans-serif;
  color: #333;
  background-color: #fff;
}
```

> [!IMPORTANT]
>
> Tener en cuenta que si el archivo CSS no contiene la regla @charset, se asumirá que el conjunto de caracteres es el mismo que el del documento HTML que hace referencia al archivo CSS. Si ya has especificado el conjunto de caracteres (charset) en la etiqueta `<meta>` del documento HTML, no es necesario repetirlo en el archivo CSS utilizando la regla @charset

## @color-profile

La regla `@color-profile` se utiliza para definir y especificar perfiles de color ICC (International Color Consortium) asociados con imágenes o elementos gráficos en una página web. Estos perfiles de color permiten que los navegadores y otros agentes de usuario representen de manera más precisa los colores en las imágenes, asegurando la consistencia en la reproducción del color en diferentes dispositivos y configuraciones.

Dentro del bloque de la regla `@color-profile`, puedes incluir varias propiedades que describan el perfil de color. Algunas de las propiedades comunes son src (fuente) y rendering-intent (intención de representación).

**Ejemplo**

```css
/* CSS */

/* Definir un perfil de color llamado "sRGB" con una intención de representación perceptual y especificando las fuentes (archivos ICC) desde las cuales se debe obtener el perfil de color. */

@color-profile sRGB {
  rendering-intent: perceptual;
  src: local("sRGB Color Space Profile.icm"), local("sRGB IEC61966-2.1");
}
```

> [!IMPORTANT]
>
> La regla @color-profile se utiliza principalmente en el contexto de imágenes y gráficos que utilizan perfiles de color ICC para asegurar una representación coherente de los colores en diferentes dispositivos.
>
> **No es algo que se use comúnmente en todos los proyectos de CSS.**

## @font-face

La regla `@font-face `en CSS se utiliza para especificar fuentes personalizadas para su uso en la presentación de un documento web. Permite a los desarrolladores web incorporar fuentes específicas que no están disponibles de manera predeterminada en la mayoría de los sistemas, dando así un mayor control sobre el aspecto tipográfico de la página.

**Sintaxis**

```css
/* CSS */

@font-face {
  font-family: "NombreDeLaFuente";
  src: url("ruta/al/archivo-de-fuente.woff2") format("woff2"), url("ruta/al/archivo-de-fuente.woff")
      format("woff");
  /* 
font-family: Especifica el nombre de la fuente que se utilizará en el resto de las reglas de estilo.
src: Indica la ubicación de los archivos de la fuente. Puede incluir rutas a archivos locales o URLs remotas. Además, se especifica el formato de la fuente utilizando format(). Esto permite al navegador seleccionar la fuente adecuada según el formato que pueda manejar.
*/
}
```

**Ejemplo**

```css
/* CSS */

/* Crear la fuente personalizada */
@font-face {
  font-family: "MiFuentePersonalizada";
  src: url("ruta/a/mi-fuente.woff2") format("woff2"), url("ruta/a/mi-fuente.woff")
      format("woff");
  font-weight: normal;
  font-style: normal;
}

/* Utilizar la fuente personalizada */
body {
  font-family: "MiFuentePersonalizada", sans-serif;
}
```

> [!NOTE]
>
> La regla `@font-face` es especialmente útil para mejorar la consistencia tipográfica y el diseño visual de un sitio web al utilizar fuentes que pueden no estar presentes en todos los sistemas.

## @font-feature-values

La regla-at CSS @font-feature-values permite a los autores usar un nombre común de font-variant-alternates para características activadas de distintas formas en OpenType. Permite simplificar el código CSS cuando se usan distintas fuentes.

**Ejemplo**

```css
/* CSS */

@font-feature-values MiFuente {
  /* Definición de nombre y características alternativas */
  @styleset {
    roman: 1; /* Estilo normal */
    italic: 2; /* Estilo cursiva */
  }

  @character-variant {
    swash: swsh; /* Variante de glifo decorativa */
    small-caps: smcp; /* Letras versalitas */
  }
}

body {
  font-family: "MiFuente", sans-serif;
  font-variant-alternates: styleset(italic), character-variant(swash);
}

/* 
Se define un conjunto de estilos (@styleset) con nombres roman e italic asociados con valores numéricos específicos.

Se define una variante de carácter (@character-variant) con nombres swash y small-caps asociados con códigos específicos.

Luego, en el selector body, se utiliza font-variant-alternates para aplicar las características alternativas usando los nombres definidos previamente en @font-feature-values.
*/
```
