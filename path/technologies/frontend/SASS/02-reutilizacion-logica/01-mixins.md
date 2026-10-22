<h1 align="center">Mixins</h1>

<h2>📑 Contenido</h2>

- [Mixins](#mixins)
- [¿Por qué usar mixins?](#por-qué-usar-mixins)
- [Cómo se definen y se usan](#cómo-se-definen-y-se-usan)
  - [Definición](#definición)
  - [Inclusión](#inclusión)
- [Mixins con parámetros](#mixins-con-parámetros)

## Mixins

Un mixin es un bloque de estilos reutilizable. Piensa en él como una plantilla que puedes definir una vez y luego "incluir" en cualquier lugar de tus hojas de estilo. Su propósito principal es evitar la repetición de código (el principio DRY: "Don't Repeat Yourself").

## ¿Por qué usar mixins?

Imagina que tienes una serie de propiedades CSS que usas una y otra vez en diferentes selectores. Por ejemplo, los prefijos de proveedor para un transition o los estilos de un flexbox. En lugar de escribir todas las líneas de código cada vez, las guardas en un mixin.

Esto hace tu código:

- Más limpio y conciso: Defines el mixin una sola vez.

- Más fácil de mantener: Si necesitas cambiar una de las propiedades, solo la modificas en el mixin y el cambio se aplica en todos los lugares donde lo hayas incluido.

## Cómo se definen y se usan

### Definición

Se usa la directiva @mixin seguida del nombre del mixin.

```scss
// Sass
@mixin flexbox-centrado {
  display: flex;
  justify-content: center;
  align-items: center;
}
```

### Inclusión

Para usar el mixin en un selector, se usa la directiva @include seguida del nombre del mixin.

```scss
// Sass
.contenedor {
  width: 100%;
  height: 200px;
  background-color: #f0f0f0;
  @include flexbox-centrado; // Aquí incluimos el mixin
}
```

El código compilado a CSS se vería así:

```css
// Sass
/* CSS compilado */
.contenedor {
  width: 100%;
  height: 200px;
  background-color: #f0f0f0;
  display: flex;
  justify-content: center;
  align-items: center;
}
```

## Mixins con parámetros

La verdadera potencia de los mixins viene de su capacidad para aceptar parámetros. Esto te permite pasar valores (como variables, colores, o números) al mixin para que sean usados en sus propiedades.

```scss
// Sass
// Definimos el mixin con un parámetro $color
@mixin boton($color) {
  background-color: $color;
  border: 1px solid darken($color, 15%);
  color: white;
  padding: 10px 20px;
  border-radius: 5px;

  &:hover {
    background-color: darken($color, 10%);
  }
}

.boton-primario {
  @include boton(#007bff); // Pasamos el color azul
}

.boton-secundario {
  @include boton(#6c757d); // Pasamos el color gris
}
```

```css
/* CSS compilado */
.boton-primario {
  background-color: #007bff;
  border: 1px solid #004d9c;
  color: white;
  padding: 10px 20px;
  border-radius: 5px;
}
.boton-primario:hover {
  background-color: #0069d9;
}

.boton-secundario {
  background-color: #6c757d;
  border: 1px solid #494f54;
  color: white;
  padding: 10px 20px;
  border-radius: 5px;
}
.boton-secundario:hover {
  background-color: #5a6268;
}
```
