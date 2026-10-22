<h1 align="center">Interpolación</h1>

<h2>📑 Contenido</h2>

- [Interpolación (#{$variable})](#interpolación-variable)
- [¿Por qué es útil?](#por-qué-es-útil)
  - [Ejemplo 1: Nombres de clases dinámicos](#ejemplo-1-nombres-de-clases-dinámicos)
  - [Ejemplo 2: Nombres de propiedades](#ejemplo-2-nombres-de-propiedades)
- [Puntos clave sobre la interpolación:](#puntos-clave-sobre-la-interpolación)

## Interpolación (#{$variable})

La interpolación en Sass es una característica poderosa que te permite incrustar el valor de una variable dentro de cadenas de texto (strings) en tus selectores, nombres de propiedades o valores. Se usa la sintaxis `#{}` para "interpolar" o "incrustar" el contenido de una variable.

## ¿Por qué es útil?

La interpolación es clave cuando necesitas generar nombres de clases, selectores o propiedades dinámicamente. Sin ella, Sass no sabría cómo interpretar un valor de variable fuera de las declaraciones de estilo normales.

### Ejemplo 1: Nombres de clases dinámicos

Imagina que tienes una serie de iconos y quieres generar clases como `.icono-usuario`, `.icono-cesta`, `.icono-ajustes` sin escribir cada una de forma manual. Puedes hacerlo con una lista y un bucle, usando la interpolación para crear el nombre de la clase.

```scss
$iconos: "usuario", "cesta", "ajustes";

@each $icono in $iconos {
  .icono-#{$icono} {
    background-image: url("../img/iconos/#{$icono}.svg");
    // Otros estilos...
  }
}
```

El código de arriba se compilará a esto:

```css
.icono-usuario {
  background-image: url("../img/iconos/usuario.svg");
}

.icono-cesta {
  background-image: url("../img/iconos/cesta.svg");
}

.icono-ajustes {
  background-image: url("../img/iconos/ajustes.svg");
}
```

### Ejemplo 2: Nombres de propiedades

También puedes usar la interpolación para generar nombres de propiedades CSS, aunque es una técnica menos común.

```scss
$estilo-borde: "border-top";
$ancho: 2px;
$tipo: solid;
$color: #333;

.mi-elemento {
  #{$estilo-borde}: #{$ancho} #{$tipo} #{$color};
}
```

El código se compilará a:

```css
.mi-elemento {
  border-top: 2px solid #333;
}
```

## Puntos clave sobre la interpolación:

- Sintaxis: Siempre se usa la sintaxis `#{}` alrededor de la variable que quieres incrustar.

- Contexto: Úsala cuando necesites que el valor de una variable se trate como texto literal. Esto es más común en selectores y nombres de propiedades, pero también funciona en valores si es necesario.

- No siempre es necesaria: En la mayoría de los casos (como en el ejemplo de las variables de colores o tamaños), simplemente usar `$mi-variable` es suficiente. La interpolación es para cuando necesitas más control sobre la estructura de lo que se genera.
