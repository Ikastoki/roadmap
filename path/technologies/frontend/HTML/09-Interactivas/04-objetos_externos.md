<h1 align="center">Objetos Externos</h1>

<h2>📑 Contenido</h2>

- [Object](#object)
- [Iframe](#iframe)
- [Embed](#embed)

## Object

La etiqueta `<object>` se utiliza para incrustar objetos en una página web, permitiendo la inclusión de varios tipos de contenido, como imágenes, audio, video, applets de Java, y otros objetos incrustables. La función principal de `<object>` es proporcionar un contenedor genérico para contenido externo que puede ser renderizado por el navegador.

```html
<!-- HTML -->

<!-- Incrustar un archivo PDF -->
<object data="archivo.pdf" type="application/pdf" width="400" height="300">
  <p>
    El contenido PDF no se puede mostrar. Puedes
    <a href="archivo.pdf">descargarlo aquí</a>.
  </p>
</object>
```

> [!NOTE]
>
> Mirar ejemplo visual de la demo en [MDN](https://developer.mozilla.org/es/docs/Web/HTML/Element/object)

## Iframe

La etiqueta `<iframe>` (Inline Frame) se utiliza para incrustar contenido externo dentro de una página web. Esta etiqueta permite mostrar otro documento HTML, una página web completa, o incluso contenido de otros dominios. Los iframes son útiles para incrustar mapas, videos de YouTube, contenido de redes sociales, widgets y otros elementos externos.

```HTML
<!-- Insertar un video de YouTube -->

<!-- Pasos:
    1. Ir a un video de YouTube
    2. Pulsar en los 3 puntes del video que quieras incrustar
    3.Compartir
    4.Insertar
    5.Copiar el código del Iframe.
    6.Pegar el código en tu página.

 -->

<iframe
    width="560"
    height="315"
    src="https://www.youtube-nocookie.com/embed/yg7gZCZ_ODk"
    title="YouTube video player"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowfullscreen
></iframe>
```

> [!IMPORTANT]
>
> Para evitar que tu página web sea incrustada o utilizada dentro de un `<iframe>`, puedes utilizar la política de seguridad del encabezado **X-Frame-Options**. Este encabezado HTTP ayuda a prevenir ataques de clickjacking al indicar a los navegadores si permitir o no que una página sea mostrada dentro de un marco (`<iframe>`).
>
> - **X-Frame-Options: DENY** no puede ser mostrada.
> - **X-Frame-Options: SAMEORIGIN** solo si la solicitud proviene de la misma origen (dominio) que la página.
> - **X-Frame-Options: ALLOW-FROM https://ejemplo.com** solo si la solicitud proviene de la URI especificada.

## Embed

La etiqueta `<embed>` se utiliza para incrustar contenido multimedia, como audio o video, en una página web. A través de esta etiqueta, puedes proporcionar una forma sencilla de incrustar contenido multimedia sin necesidad de plugins adicionales.

```html
<!-- HTML -->

<h2>Video incrustado</h2>
<embed src="video.mp4" type="video/mp4" width="400" height="300" />
```

> [!IMPORTANT]
>
> Es importante tener en cuenta que mientras `<embed>` es una forma de incrustar contenido multimedia, en el caso específico de videos, se prefiere el uso de la etiqueta [`<video>`](../08-Multimedia/03-video.md) para tener más control sobre la reproducción y ofrecer una mejor compatibilidad con los navegadores modernos.


