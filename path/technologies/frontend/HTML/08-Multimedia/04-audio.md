<h1 align="center">Audio</h1>

<h2>📑 Contenido</h2>

- [Audio](#audio)
- [Track](#track)

## Audio

El elemento `<audio>` funciona de la misma manera que el elemento video, comparten la gran mayoría de atributos.

```html
<!-- Ejemplo Audio -->
<audio controls>
  <source src="ruta/del/audio.ogg" type="audio/ogg" />
  <source src="ruta/del/audio.mp3" type="audio/mpeg" />
</audio>
```

## Track

La etiqueta `<track>` se usa para añadir pistas de texto a elementos de medios como `<video>` y `<audio>`. Las pistas de texto pueden ser subtítulos, títulos, descripciones, capítulos o metadatos.

**Atributos más usados:**

- **src:** Especifica la ruta del video.
- **srclang:** Código de idioma ISO 639-1 de los subtítulos.
- **label:** Título que verá el usuario para elegir al pulsar en el botón CC.
- **kind:** Indica el tipo o género de subtítulos enlazados
- **default(boolean):** Marca este canal como los subtítulos principales por defecto.

```html
<!-- Ejemplo Track -->
<video width="320" height="240" controls>
  <source src="mi-video.mp4" type="video/mp4" />
  <source src="mi-video.ogg" type="video/ogg" />
  <track src="mi-sub.vtt" kind="subtitles" srclang="en" label="English" />
  <track src="mi-sub.vtt" kind="subtitles" srclang="es" label="Español" />
</video>
```


