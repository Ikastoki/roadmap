<h1 align="center">Partials</h1>

<h2>📑 Contenido</h2>

- [Partials (Archivos Parciales)](#partials-archivos-parciales)
- [¿Por qué usar partials?](#por-qué-usar-partials)
- [Estructura típica de un proyecto Sass](#estructura-típica-de-un-proyecto-sass)
- [¿Cómo se usan? (@import y @use)](#cómo-se-usan-import-y-use)
  - [El sistema `@import` (Antiguo, pero aún común)](#el-sistema-import-antiguo-pero-aún-común)
  - [El sistema `@use` (Recomendado y más moderno)](#el-sistema-use-recomendado-y-más-moderno)
  - [Diferencia:](#diferencia)

## Partials (Archivos Parciales)

Los partials son archivos de Sass que se nombran comenzando con un guion bajo (`_`), por ejemplo, `_variables.scss` o `_botones.scss`. El guion bajo le indica a Sass que estos archivos no deben compilarse directamente a archivos CSS separados. En cambio, su propósito es ser importados en un archivo Sass principal.

## ¿Por qué usar partials?

La razón principal para usar partials es la organización y modularización de tu código. En lugar de tener un único y gigantesco archivo styles.scss, puedes dividir tus estilos en archivos más pequeños y manejables, cada uno con una función específica.

Esto hace que tu código sea:

- Más fácil de mantener: Es mucho más sencillo encontrar y editar los estilos de un componente específico (como un botón o una tarjeta) cuando están en su propio archivo.

- Más legible: Un archivo principal limpio que importa todos los partials es mucho más fácil de entender de un vistazo.

- Más reutilizable: Puedes crear partials de "utilidades" o "configuración" que puedes importar en múltiples proyectos.

## Estructura típica de un proyecto Sass

Una estructura común de carpetas para un proyecto Sass se ve así:

```
sass/
|-- abstracts/
|   |-- _variables.scss  (variables globales)
|   |-- _mixins.scss     (mixins)
|   |-- _functions.scss  (funciones)
|-- components/
|   |-- _buttons.scss
|   |-- _cards.scss
|-- layout/
|   |-- _header.scss
|   |-- _footer.scss
|-- pages/
|   |-- _home.scss
|-- base/
|   |-- _base.scss       (estilos base como `body`, `html`)
|-- main.scss            (el archivo principal que lo importa todo)
```

En este ejemplo, `main.scss` es el único archivo que se compilará a `main.css`.

## ¿Cómo se usan? (@import y @use)

Para incluir un partial en otro archivo Sass, se utiliza la directiva `@import` o la más moderna `@use`.

### El sistema `@import` (Antiguo, pero aún común)

Simplemente usas `@import` seguido del nombre del partial, sin el guion bajo y sin la extensión `.scss`.

```scss
// main.scss

@import "abstracts/variables";
@import "abstracts/mixins";
@import "components/buttons";
@import "layout/header";

body {
  // Estilos del body
}
```

### El sistema `@use` (Recomendado y más moderno)

Sass ha introducido el sistema de `@use` como un reemplazo para `@import`. Este sistema es más explícito y evita conflictos de nombres. Cuando usas `@use`, los estilos del partial se cargan en un namespace (espacio de nombres).

```scss
// main.scss

@use "abstracts/variables" as *; // El `as *` importa las variables sin un namespace
@use "components/buttons";

body {
  background-color: $color-primario; // Si se importó con 'as *'
}

.mi-boton {
  @include buttons.boton-estilo-primario; // Se accede a mixins y funciones a través del namespace `buttons`
}
```

### Diferencia:

- `@import`: Carga todo en el espacio global. Es más simple, pero puede causar conflictos. Sass planea eliminarlo en el futuro.

- `@use`: Carga el contenido en un namespace. Es más seguro y explícito, y es la práctica recomendada hoy en día.
