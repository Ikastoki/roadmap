<h1 align="center">Bucles</h1>

<h2>📑 Contenido</h2>

- [Bucles](#bucles)
- [Bucle @for](#bucle-for)
- [Bucle @each](#bucle-each)
- [Bucle @while](#bucle-while)

## Bucles

Los bucles en Sass te permiten generar estilos repetitivos de forma automática. En lugar de escribir el mismo bloque de código una y otra vez para cada variación (por ejemplo, para cada columna de un grid o para cada tamaño de fuente), puedes usar un bucle para que Sass lo haga por ti. Esto es ideal para la automatización y la creación de sistemas de diseño escalables.

## Bucle @for

El bucle `@for` se utiliza para repetir un bloque de código un número específico de veces. Es ideal para crear cosas como clases de utilidades o columnas de una cuadrícula.

Hay dos formas de usarlo:

`through`: Incluye el último número en el bucle. desde 1 hasta 3 incluiría 1, 2, y 3.

`to`: Excluye el último número del bucle. desde 1 hasta 3 incluiría 1 y 2.

```scss
// Sass
@for $i from 1 through 3 {
  .tamano-#{$i} {
    // Usamos interpolación para el nombre de la clase
    font-size: $i * 0.5rem + 1rem;
  }
}
```

```css
/* CSS compilado */
.tamano-1 {
  font-size: 1.5rem;
}

.tamano-2 {
  font-size: 2rem;
}

.tamano-3 {
  font-size: 2.5rem;
}
```

## Bucle @each

El bucle `@each` es la forma más común y poderosa de bucle en Sass. Itera sobre cada elemento en una lista o un mapa, lo que te permite generar estilos basados en los datos que defines.

Ejemplo: Generar clases de colores para texto.

```scss
// Sass
$colores-principales:
  azul: #007bff,
  verde: #28a745,
  rojo: #dc3545;

@each $nombre, $color in $colores-principales {
  .texto-#{$nombre} {
    color: $color;
  }
}
```

En este caso, `@each` itera sobre el mapa `$colores-principales`, y en cada iteración, `$nombre` y `$color` se asignan al par clave-valor correspondiente.

CSS compilado:

```css
/* CSS compilado */
.texto-azul {
  color: #007bff;
}

.texto-verde {
  color: #28a745;
}

.texto-rojo {
  color: #dc3545;
}
```

## Bucle @while

El bucle `@while` repite un bloque de código mientras una condición es verdadera. Aunque es útil, es menos común que `@for` o `@each` y debes tener cuidado de no crear un bucle infinito.

```scss
// Sass
$i: 1;

@while $i < 4 {
  .item-#{$i} {
    width: 20px * $i;
  }
  $i: $i + 1; // Incrementamos el valor de $i para que el bucle no sea infinito
}
```

```scss
/* CSS compilado */
.item-1 {
  width: 20px;
}

.item-2 {
  width: 40px;
}

.item-3 {
  width: 60px;
}
```
