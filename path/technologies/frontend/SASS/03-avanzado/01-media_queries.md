<h1 align="center">Media queries (con mixins)</h1>

<h2>📑 Contenido</h2>

- [Media queries (con mixins)](#media-queries-con-mixins)
- [¿Cómo funciona?](#cómo-funciona)
- [Definir los puntos de ruptura (breakpoints)](#definir-los-puntos-de-ruptura-breakpoints)
- [Crear el mixin de media query](#crear-el-mixin-de-media-query)
- [Usar el mixin en tu código](#usar-el-mixin-en-tu-código)

## Media queries (con mixins)

Escribir media queries directamente en CSS puede volverse repetitivo y desordenado, especialmente en proyectos grandes. Sass resuelve este problema permitiéndote encapsular toda la lógica de tus media queries dentro de un mixin, lo que hace tu código más limpio, legible y reutilizable.

## ¿Cómo funciona?

En lugar de escribir la media query completa cada vez que necesitas estilos responsivos, defines un mixin que toma el tamaño de pantalla como parámetro. Dentro del mixin, escribes la regla @media y luego incluyes la lógica de tu CSS.

## Definir los puntos de ruptura (breakpoints)

Una buena práctica es guardar los tamaños de pantalla comunes en un mapa de Sass. Esto centraliza tus breakpoints en un solo lugar, facilitando los cambios globales en el futuro.

```scss
// _variables.scss o _config.scss
$breakpoints: (
  "movil": 576px,
  "tablet": 768px,
  "escritorio": 992px,
);
```

## Crear el mixin de media query

Ahora, creamos un mixin que toma un nombre de breakpoint como argumento y lo usa para generar la media query.

```scss
// _mixins.scss
@mixin responsive($breakpoint) {
  // Verificamos si el breakpoint existe en nuestro mapa
  @if map-has-key($breakpoints, $breakpoint) {
    // Si existe, lo usamos para generar la media query
    @media (min-width: map-get($breakpoints, $breakpoint)) {
      @content; // Esta es la magia: @content inserta el código que le pases
    }
  } @else {
    // Si no existe, lanzamos un error para evitar problemas
    @error "Breakpoint '#{$breakpoint}' no encontrado en el mapa de breakpoints.";
  }
}
```

La directiva `@content` es la clave aquí. Es un marcador de posición donde Sass insertará el bloque de código CSS que le pases cuando uses el mixin.

## Usar el mixin en tu código

Ahora, en cualquier parte de tu hoja de estilo, puedes usar el mixin para aplicar estilos condicionales de forma sencilla.

```scss
// _componentes.scss

.tarjeta-producto {
  width: 100%;
  padding: 1rem;

  // Incluimos el mixin para el breakpoint 'tablet'
  @include responsive("tablet") {
    // Todos los estilos aquí dentro se aplicarán solo a partir de 768px
    width: 50%;
    float: left;
    padding: 2rem;
  }
}
```

```scss
/* CSS compilado */
.tarjeta-producto {
  width: 100%;
  padding: 1rem;
}
@media (min-width: 768px) {
  .tarjeta-producto {
    width: 50%;
    float: left;
    padding: 2rem;
  }
}
```
