<h1 align="center">Agrupar Información</h1>

<h2>📑 Contenido</h2>

- [Agrupar información(contactos)](#agrupar-informacióncontactos)
- [Citar](#citar)
- [Fecha y Hora](#fecha-y-hora)

## Agrupar información(contactos)

La etiqueta `<address>` se utiliza para marcar información de contacto o detalles de contacto, como direcciones físicas, direcciones de correo electrónico o números de teléfono.

> [!NOTE]
> La etiqueta `<address> `no impone ningún formato o estilo específico, pero se considera una etiqueta semántica que ayuda a los motores de búsqueda y a los navegadores a reconocer la información de contacto.

```html
<!-- HTML -->
<address>
  <p>Nombre de la Empresa</p>
  <p>Dirección: 123 Calle Principal, Ciudad, País</p>
  <p>Teléfono: <a href="tel:+123456789">+123 456 789</a></p>
  <p>
    Correo Electrónico: <a href="mailto:info@empresa.com">info@empresa.com</a>
  </p>
</address>
```

## Citar

Para utilizar citas, fuentes y referencias en HTML hay tres etiquetas principales`<blockquote>,<q>,<cite>`.

- `<blockquote>` Para insertar una cita en forma de bloques.
- `<q>` Para insertar una cita en forma de línea.
- `<cite>` Para insertar una cita que haga referencia a un trabajo creativo. Se suele usar dentro del `<blockquote>` para informar sobre la fuente de la cita.

```HTML
<!-- Ejemplo -->
    <blockquote cite="https://developer.mozilla.org/es/">
      <p>
        <strong>Info:</strong> Página web con una gran cantidad de documentación
        sobre HTML,CSS,JS...
      </p>
    </blockquote>
```

## Fecha y Hora

La etiqueta `<time>` se utiliza para marcar contenido que representa información de fecha y hora. Puede utilizarse para mostrar fechas, horas, marcas de tiempo, duraciones y otros datos relacionados con el tiempo. La etiqueta en sí misma no muestra nada directamente en la página; su propósito principal es proporcionar información estructurada y semántica para ayudar a los navegadores y a otros agentes de usuario a interpretar y presentar la información temporal de manera adecuada.

La etiqueta `<time>` tiene un atributo `datetime` que se utiliza para proporcionar una representación en formato de fecha y hora que sea legible por máquinas, lo que ayuda a la identificación precisa del tiempo.

> [!NOTE]
>
> El atributo datetime contiene la fecha en formato ISO 8601 (AAAA-MM-DD)

```html
<p>
  La fecha de publicación de este artículo es
  <time datetime="2050-10-25">25 de octubre de 2050</time>.
</p>
```

