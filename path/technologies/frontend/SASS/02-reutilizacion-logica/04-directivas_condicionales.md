<h1 align="center">Directivas Condicionales</h1>

<h2>📑 Contenido</h2>

- [Directivas Condicionales](#directivas-condicionales)
- [Cómo funciona @if](#cómo-funciona-if)
- [Cómo funciona @else y @else if](#cómo-funciona-else-y-else-if)
- [Uso práctico con mixins](#uso-práctico-con-mixins)

## Directivas Condicionales

Las directivas condicionales te permiten aplicar estilos solo cuando se cumple una condición. Son como las declaraciones `if/else` que encuentras en los lenguajes de programación, lo que te da un control mucho más granular sobre cómo se generan tus estilos.

## Cómo funciona @if

La directiva @if evalúa una expresión. Si la expresión es true, el bloque de estilos dentro del @if se compilará a CSS. Si es false, se ignorará.

```scss
// Sass
$tema: "dark";

body {
  @if $tema == "dark" {
    background-color: #333;
    color: white;
  }
}
```

En este caso, como `$tema` es igual a 'dark', el CSS compilado incluirá los estilos para el fondo oscuro.

## Cómo funciona @else y @else if

Puedes encadenar múltiples condiciones usando `@else if` y `@else`. El bloque `@else if` solo se evalúa si el `@if` anterior es `false`, y el bloque `@else` es el último recurso que se ejecuta si ninguna de las condiciones anteriores es `true`.

```scss
// Sass
$tamanio-fuente: 18px;

.texto {
  @if $tamanio-fuente > 20px {
    font-size: 2rem;
  } @else if $tamanio-fuente == 18px {
    font-size: 1.5rem;
  } @else {
    font-size: 1rem;
  }
}
```

CSS compilado:

Como `$tamanio-fuente` es 18px, la segunda condición se cumple. El CSS generado será:

```css
/* CSS compilado */
.texto {
  font-size: 1.5rem;
}
```

## Uso práctico con mixins

Las directivas condicionales son especialmente poderosas cuando se usan dentro de mixins para añadir lógica. Por ejemplo, un mixin que genera estilos para botones podría tener un parámetro que determine el tamaño.

```scss
// Sass
@mixin boton-estilos($tamano) {
  @if $tamano == "grande" {
    padding: 15px 30px;
    font-size: 1.25rem;
  } @else if $tamano == "pequeno" {
    padding: 5px 10px;
    font-size: 0.875rem;
  } @else {
    // Por defecto
    padding: 10px 20px;
    font-size: 1rem;
  }
}

.boton-de-accion {
  @include boton-estilos("grande");
}
```

CSS compilado:

```css
/* CSS compilado */
.boton-de-accion {
  padding: 15px 30px;
  font-size: 1.25rem;
}
```
