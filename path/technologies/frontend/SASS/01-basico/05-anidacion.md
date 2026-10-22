<h1 align="center">Anidación</h1>

<h2>📑 Contenido</h2>

- [Anidación (Nesting) nesting-a-sintaxis-limpia nesting-con-selectores-de-clase](#anidación-nesting-nesting-a-sintaxis-limpia-nesting-con-selectores-de-clase)
- [¿Cómo funciona?](#cómo-funciona)
  - [Ejemplo 1: Selectores de clase anidados](#ejemplo-1-selectores-de-clase-anidados)
- [Selector \& (El Ampersand)](#selector--el-ampersand)

## Anidación (Nesting) nesting-a-sintaxis-limpia nesting-con-selectores-de-clase

La anidación en Sass te permite escribir reglas de CSS una dentro de otra. Esto sigue la misma jerarquía visual de tu HTML, haciendo tu código mucho más legible y organizado. En lugar de repetir los selectores parentales una y otra vez, simplemente los anidas.

> [!IMPORTANT]
>
> CSS ya permite la anidación.

## ¿Cómo funciona?

En Sass, puedes anidar selectores de elementos, clases, IDs e incluso pseudo-clases y pseudo-elementos. El selector interno se convierte en un hijo del selector externo.

### Ejemplo 1: Selectores de clase anidados

Imagina que tienes este HTML:

```html
<nav class="navegacion">
  <ul>
    <li><a href="#">Inicio</a></li>
    <li><a href="#">Servicios</a></li>
  </ul>
</nav>
```

En CSS tradicional, tendrías que escribir algo así:

```css
.navegacion {
  font-family: "Arial", sans-serif;
}
.navegacion ul {
  list-style: none;
  padding: 0;
}
.navegacion li {
  display: inline-block;
}
.navegacion a {
  color: #333;
  text-decoration: none;
}
```

Con la anidación de Sass, puedes escribirlo de una forma mucho más limpia y compacta:

```scss
// Sass
.navegacion {
  font-family: "Arial", sans-serif;
  ul {
    list-style: none;
    padding: 0;
    li {
      display: inline-block;
      a {
        color: #333;
        text-decoration: none;
      }
    }
  }
}
```

Como puedes ver, la estructura del CSS anidado se parece mucho más a la estructura de tu HTML, lo que facilita la lectura y el mantenimiento.

## Selector & (El Ampersand)

El ampersand (`&`) es un selector especial en Sass que representa el selector padre. Es increíblemente útil para referenciar el selector padre en el mismo nivel de anidación.

Uso con pseudo-clases y pseudo-elementos:

```scss
// Sass
.boton {
  background-color: #007bff;
  color: white;
  padding: 10px 20px;

  &:hover {
    // Esto se compila a .boton:hover
    background-color: darken(#007bff, 10%);
  }

  &:active {
    // Esto se compila a .boton:active
    background-color: darken(#007bff, 20%);
  }
}
```

Uso con clases relacionadas (BEM):

```scss
// Sass
.card {
  border: 1px solid #ddd;
  padding: 1rem;

  &__titulo {
    // Se compila a .card__titulo
    font-size: 1.5rem;
    color: #333;
  }

  &__imagen {
    // Se compila a .card__imagen
    max-width: 100%;
  }
}
```

> [!CAUTION]
>
> Cuidado con la anidación excesiva
>
> Aunque la anidación es poderosa, es importante no abusar de ella. Anidar demasiados niveles puede generar selectores de CSS muy largos y específicos, lo que dificulta la anulación de estilos y puede hacer que tu código sea menos eficiente. Una buena práctica es no anidar más de 3 o 4 niveles de profundidad.
