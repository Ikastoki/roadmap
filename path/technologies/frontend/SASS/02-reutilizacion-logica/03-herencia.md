<h1 align="center">Herencia</h1>

<h2>📑 Contenido</h2>

- [Herencia (@extend)](#herencia-extend)
- [¿Cómo funciona?](#cómo-funciona)
- [Ejemplo de uso](#ejemplo-de-uso)
- [Mixins vs. @extend: ¿Cuál usar y cuándo?](#mixins-vs-extend-cuál-usar-y-cuándo)

## Herencia (@extend)

La herencia en Sass, a través de la directiva `@extend`, te permite compartir un conjunto de propiedades CSS entre selectores. En lugar de copiar y pegar los mismos estilos, un selector puede "extender" o heredar los estilos de otro.

## ¿Cómo funciona?

Cuando un selector usa `@extend` para heredar los estilos de otro, Sass agrupa ambos selectores en el CSS final. Esto significa que el código no se duplica. En cambio, Sass crea un selector de lista que incluye tanto el selector original como el que hereda sus estilos.

## Ejemplo de uso

Imagina que tienes una clase base para un botón y quieres crear diferentes tipos de botones (primario, secundario) que compartan los mismos estilos base.

```scss
// Sass
.boton-base {
  display: inline-block;
  padding: 10px 20px;
  border-radius: 5px;
  font-weight: bold;
}

.boton-primario {
  @extend .boton-base; // Aquí extendemos los estilos base
  background-color: #007bff;
  color: white;
}

.boton-secundario {
  @extend .boton-base; // Y aquí también
  background-color: #6c757d;
  color: white;
}
```

CSS Compilado:

```css
/* CSS compilado */
.boton-base,
.boton-primario,
.boton-secundario {
  display: inline-block;
  padding: 10px 20px;
  border-radius: 5px;
  font-weight: bold;
}

.boton-primario {
  background-color: #007bff;
  color: white;
}

.boton-secundario {
  background-color: #6c757d;
  color: white;
}
```

Como puedes ver, Sass agrupó los tres selectores porque todos comparten los mismos estilos. Esto resulta en un CSS más compacto y eficiente.

## Mixins vs. @extend: ¿Cuál usar y cuándo?

Esta es otra pregunta clave. Aunque ambos se usan para reutilizar código, hay diferencias importantes:

| Característica        | `@extend`                                                                     | `@mixin`                                                                              |
| --------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **Propósito**         | Compartir estilos entre selectores.                                           | Reutilizar bloques de CSS, incluso con diferentes valores.                            |
| **Generación de CSS** | Agrupa los selectores para evitar duplicación.                                | Genera un nuevo bloque de CSS para cada inclusión.                                    |
| **Mejor para**        | Estilos de componentes con relación "es un" (ej. botón primario es un botón). | Reutilizar código con parámetros (ej. prefijos de flexbox, estilos de media queries). |

En general, la mayoría de los desarrolladores prefieren usar mixins para la reutilización de código. Aunque @extend puede generar un CSS más pequeño en algunos casos, su comportamiento puede ser impredecible y llevar a selectores muy complejos. Los mixins son más explícitos y seguros.
