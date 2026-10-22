<h1 align="center">Favicon</h1>

<h2>📑 Contenido</h2>

- [Favicon](#favicon)
- [Detalles a considerar](#detalles-a-considerar)
- [Múltiples tamaños y formatos](#múltiples-tamaños-y-formatos)
  - [Beneficios](#beneficios)
  - [Decisión del navegador](#decisión-del-navegador)
  - [Comportamiento en dispositivos específicos](#comportamiento-en-dispositivos-específicos)

## Favicon

El favicon (abreviatura de "favorite icon") es un pequeño ícono gráfico asociado a un sitio web que aparece en varios lugares, como en la pestaña del navegador, en la barra de marcadores (favoritos), y en la lista de historial. El favicon ayuda a los usuarios a identificar y acceder rápidamente a un sitio web.

> [!NOTE]
>
> Existen herramientas como [realfavicongenerator](https://realfavicongenerator.net/) para generar Favicons.

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Mi Página Web</title>
    <link rel="icon" href="ruta/al/favicon.ico" type="image/x-icon" />
    <!-- También puedes usar otros formatos de imagen como PNG -->
    <!-- <link rel="icon" href="ruta/al/favicon.png" type="image/png"> -->
  </head>
  <body>
    <h1>Bienvenido a mi página web</h1>
  </body>
</html>
```

## Detalles a considerar

- **Formato del archivo:** El formato tradicional para favicons es .ico, pero los navegadores modernos también aceptan formatos como .png, .jpg, .svg, entre otros.

- **Tamaño del ícono:** Los tamaños comunes son 16x16 píxeles y 32x32 píxeles, aunque también puedes incluir versiones más grandes para dispositivos con pantallas de alta resolución.

- **Ubicación del archivo:** Es común colocar el archivo del favicon en la raíz del directorio del sitio web para facilitar su acceso y referencia.

## Múltiples tamaños y formatos

Es una buena práctica incluir múltiples tamaños y formatos de favicon. Hacerlo asegura que el favicon se vea bien en diferentes contextos y dispositivos.

### Beneficios

- **Compatibilidad con diferentes navegadores y dispositivos:** Diferentes navegadores y dispositivos pueden tener diferentes requisitos y preferencias para favicons. Incluir varios tamaños y formatos asegura que tu favicon se verá correctamente en todos ellos.

- **Optimización para alta resolución:** Los dispositivos con pantallas de alta resolución (como los dispositivos Apple Retina) se benefician de íconos más grandes y detallados. Proveer favicons de diferentes tamaños ayuda a que se vean nítidos y claros en cualquier pantalla.

- **Flexibilidad en el diseño:** Al incluir múltiples tamaños, te aseguras de que tu favicon se vea bien tanto en contextos pequeños (como una pestaña de navegador) como en contextos más grandes (como la pantalla de inicio de un dispositivo móvil).

```html
<head>
  <link
    rel="icon"
    sizes="16x16"
    href="ruta/al/favicon-16x16.png"
    type="image/png"
  />
  <link
    rel="icon"
    sizes="32x32"
    href="ruta/al/favicon-32x32.png"
    type="image/png"
  />
  <link
    rel="icon"
    sizes="48x48"
    href="ruta/al/favicon-48x48.png"
    type="image/png"
  />
  <link
    rel="icon"
    sizes="64x64"
    href="ruta/al/favicon-64x64.png"
    type="image/png"
  />
</head>
```

### Decisión del navegador

El navegador selecciona el tamaño o formato de favicon adecuado en función del contexto en el que se utiliza, así como las especificaciones proporcionadas en las etiquetas `<link>` del documento HTML. Los atributos sizes y type en la etiqueta `<link>` ayudan al navegador a determinar qué favicon usar.

Si un favicon de 16x16 píxeles se necesita para una pestaña del navegador, el navegador buscará una etiqueta `<link>` con sizes="16x16".

El navegador puede también considerar el atributo type para asegurarse de que usa el formato de imagen compatible, como image/png o image/x-icon.

### Comportamiento en dispositivos específicos

- **Dispositivos Apple (iOS):** Los dispositivos iOS buscan íconos específicos definidos con la etiqueta rel="apple-touch-icon" y seleccionan el tamaño adecuado para la pantalla del dispositivo.

  ```html
  <link
    rel="apple-touch-icon"
    sizes="180x180"
    href="ruta/al/apple-icon-180x180.png"
  />
  ```

- **Dispositivos Android:** Para los dispositivos Android, puedes especificar favicons en diferentes tamaños para que se usen como iconos de la aplicación web.

  ```html
  <link
    rel="icon"
    sizes="192x192"
    href="ruta/al/android-icon-192x192.png"
    type="image/png"
  />
  ```
