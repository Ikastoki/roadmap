<h1 align="center">Variables</h1>

<h2>📑 Contenido</h2>

- [Variables en Sass](#variables-en-sass)
- [Cómo se declaran](#cómo-se-declaran)
- [Cómo se usan](#cómo-se-usan)

## Variables en Sass

Las variables en Sass son, como su nombre lo indica, contenedores para guardar información que puedes reutilizar en tus hojas de estilo. Piensa en ellas como atajos. En lugar de escribir el mismo valor una y otra vez, lo guardas en una variable y luego usas el nombre de esa variable cada vez que lo necesitas.

Esto es increíblemente útil para mantener la consistencia en tu diseño y para hacer cambios globales de manera rápida y eficiente. Por ejemplo, si decides cambiar el color principal de tu sitio web, solo tienes que modificar la variable una vez, y el cambio se aplicará automáticamente en todos los lugares donde la hayas usado.

## Cómo se declaran

Para declarar una variable en Sass, simplemente usas el signo de dólar (`$`) seguido del nombre de la variable y su valor.

```scss
$color-primario: #007bff;
$fuente-principal: "Arial", sans-serif;
$espaciado-base: 1rem;
```

## Cómo se usan

Una vez que has declarado una variable, puedes usarla en cualquier parte de tu archivo Sass, como si fuera el valor que contiene.

```scss
body {
  font-family: $fuente-principal;
  background-color: #f8f9fa;
}

h1 {
  color: $color-primario;
  font-size: 2.5rem;
}

.boton {
  background-color: $color-primario;
  padding: $espaciado-base;
  border-radius: 5px;
}
```

Al compilar este código Sass, se generará el siguiente CSS:

```css
body {
  font-family: "Arial", sans-serif;
  background-color: #f8f9fa;
}

h1 {
  color: #007bff;
  font-size: 2.5rem;
}

.boton {
  background-color: #007bff;
  padding: 1rem;
  border-radius: 5px;
}
```
