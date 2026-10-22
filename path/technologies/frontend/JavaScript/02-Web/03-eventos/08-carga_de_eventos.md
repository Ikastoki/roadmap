<h1 align="center">Carga de eventos</h1>

<h2>📑 Contenido</h2>

- [Carga de eventos](#carga-de-eventos)
- [Tipos de Eventos de Carga](#tipos-de-eventos-de-carga)
  - [DOMContentLoaded](#domcontentloaded)
  - [load](#load)
- [Diferencias](#diferencias)

## Carga de eventos

El concepto de carga de eventos se refiere a cómo y cuándo se ejecutan los event listeners relacionados con la carga del documento y sus recursos.

## Tipos de Eventos de Carga

Existen varios eventos que puedes usar para controlar cuándo se ejecuta tu código. Los más comunes son DOMContentLoaded y load.

### DOMContentLoaded

¿Qué hace?

Este evento se dispara tan pronto como el documento HTML ha sido completamente cargado y parseado por el navegador. El árbol DOM ya está construido y es seguro manipularlo.

¿Qué no hace?

No espera a que se carguen recursos externos como imágenes, hojas de estilo (`.css`) o scripts (`.js`) externos (a menos que no tengan el atributo `async` o `defer`).

Uso:

Es la forma recomendada de adjuntar la mayoría de tus event listeners y ejecutar el código que manipula el DOM, ya que se dispara mucho antes que el evento load, haciendo que tu página se sienta más rápida.

```javascript
document.addEventListener("DOMContentLoaded", function () {
  console.log("El DOM está listo para ser manipulado.");
  // Aquí puedes acceder a cualquier elemento de tu HTML, como un botón o un div
  const miElemento = document.getElementById("mi-elemento");
});
```

### load

¿Qué hace?:

Este evento se dispara cuando el documento HTML y todos sus recursos dependientes (imágenes, hojas de estilo, scripts, etc.) han terminado de cargarse.

¿Qué no hace?:

No se activa hasta que absolutamente todo está cargado.

Uso:

Úsalo solo cuando tu código necesita trabajar con recursos externos que tienen que estar cargados, como el tamaño de una imagen o un video.

```javascript
window.addEventListener("load", function () {
  console.log("Todos los recursos de la página están cargados.");
  // Ahora es seguro obtener el tamaño real de una imagen, por ejemplo
  const miImagen = document.getElementById("mi-imagen");
  console.log(`El ancho de la imagen es: ${miImagen.width}px`);
});
```

En el pasado, era común usar `window.onload`, pero `DOMContentLoaded` es preferible en la mayoría de los casos por su velocidad.

## Diferencias

| Característica    | DOMContentLoaded         | load                                            |
| ----------------- | ------------------------ | ----------------------------------------------- |
| Cuándo se dispara | Cuando el DOM está listo | Cuando el DOM y todos sus recursos están listos |
| Recursos externos | No espera por ellos      | Sí espera por ellos                             |
| Velocidad         | Más rápido               | Más lento                                       |
| Uso recomendado   | Manipulación del DOM     | Tareas que dependen de recursos externos        |
