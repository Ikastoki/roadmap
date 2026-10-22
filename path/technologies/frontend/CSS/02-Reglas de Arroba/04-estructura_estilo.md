<h1 align="center">Estructura y Estilo</h1>

<h2>📑 Contenido</h2>

- [@container](#container)
- [@counter-style](#counter-style)
- [@keyframes](#keyframes)
- [@page](#page)

## @container

La regla `@container` son el mismo concepto que las [Media Queries](../03-Layouts/01-Basicos/07-media_query.md), pero orientados a modificar los estilos dependiendo del tamaño de la página o dispositivo, lo hace dependiendo de un contenedor padre específico. Se puede cambiar el tamaño de ciertos elementos en función del contexto donde se encuentre

## @counter-style

Se utiliza para definir estilos personalizados para **contadores** de lista y numeración en documentos. Permite personalizar la apariencia y el formato de los números o caracteres utilizados en listas numeradas.

**Propiedades:**

- **system:**Especifica el sistema de numeración a utilizar. Algunos valores posibles son:
  - `decimal:` Sistema decimal (por defecto).
  - `cyclic:` Sistema cíclico.
  - `fixed:` Sistema fijo.
- **symbols:** Define los símbolos que se utilizarán para representar los números en el contador. Puedes usar cadenas de texto, caracteres Unicode, o incluso imágenes.
- **additive-symbols:** Especifica símbolos adicionales que se agregarán a los números cuando haya un desbordamiento en el contador.
- **suffix y prefix:** Agrega un sufijo o prefijo a cada número del contador.
- **range:** Define un rango específico de valores para el contador.

```html
<!-- HTML -->
<h1>Lista Numerada con Counter-Style Personalizado</h1>

<ol>
  <li>Elemento 1</li>
  <li>Elemento 2</li>
  <li>Elemento 3</li>
</ol>
```

```css
/* CSS */
@counter-style custom-counter {
  system: fixed;
  symbols: "🔴" "🔵" "🟢";
  suffix: " ";
}

ol {
  counter-reset: custom-counter;
  list-style: custom-counter;
}

li {
  margin-bottom: 10px;
}
```

## @keyframes

Se utiliza para definir animaciones. Con @keyframes, puedes especificar cómo cambian los estilos de un elemento durante una animación a lo largo del tiempo.

**Sintaxis básica:**

```css
/* CSS */
@keyframes nombreDeAnimacion {
  from {
    /* Estilos iniciales */
  }
  to {
    /* Estilos finales */
  }
}

/* O también puedes especificar puntos intermedios usando porcentaje */
@keyframes nombreDeAnimacion {
  0% {
    /* Estilos iniciales */
  }
  50% {
    /* Estilos a la mitad de la animación */
  }
  100% {
    /* Estilos finales */
  }
}

/* Aplicar la animación a un elemento */
.elemento {
  animation: nombreDeAnimacion 3s ease-in-out;
}
```

> [!NOTE]
>
> Ver más en el apartado para [Animaciones](../04-Animaciones%20y%20Filtros/02-Animaciones.md).

## @page

Se utiliza para definir los estilos de una página impresa cuando se realiza una impresión del contenido web. Permite controlar la apariencia de las páginas impresas, como el diseño, márgenes, encabezados, pies de página y otros aspectos específicos de la impresión.

```css
/* CSS */
@page {
  size: A4; /* Tamaño del papel, puede ser 'letter', 'A4', etc. */
  margin: 1cm; /* Márgenes de la página */
}

/* Estilos específicos para el encabezado de la página */
@page {
  margin-top: 20mm;
  margin-bottom: 10mm;
  @top-left {
    content: "Encabezado izquierdo";
  }
  @top-center {
    content: "Encabezado centrado";
  }
  @top-right {
    content: "Encabezado derecho";
  }
}

/* Estilos específicos para el pie de la página */
@page {
  margin-bottom: 20mm;
  @bottom-left {
    content: "Pie izquierdo";
  }
  @bottom-center {
    content: "Pie centrado";
  }
  @bottom-right {
    content: "Pie derecho";
  }
}
```
