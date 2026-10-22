<h1 align="center">Módulos</h1>

<h2>📑 Contenido</h2>

- [Módulos (Sistema @use de Sass)](#módulos-sistema-use-de-sass)
- [¿Por qué @use es mejor que @import?](#por-qué-use-es-mejor-que-import)
- [Cómo se usa @use](#cómo-se-usa-use)
- [Resumen del sistema de módulos](#resumen-del-sistema-de-módulos)

## Módulos (Sistema @use de Sass)

El sistema de módulos de Sass, introducido en las versiones más recientes, es la evolución del sistema @import. Su objetivo es hacer tus hojas de estilo más predecibles y fáciles de mantener, especialmente en proyectos grandes.

La directiva clave de este sistema es @use, que ya hemos mencionado brevemente. @use reemplaza el comportamiento de @import de una manera más segura y organizada.

## ¿Por qué @use es mejor que @import?

- Nombres de variables aislados (namespaces): Con @import, todas las variables, mixins y funciones se cargan en el mismo "espacio global", lo que puede causar conflictos de nombres si importas dos archivos con una variable llamada, por ejemplo, $color. Con @use, el contenido del archivo se carga en un namespace basado en el nombre del archivo.

  - @use 'botones';

  - Para acceder a un mixin o variable del archivo \_botones.scss, usarías botones.nombre-del-mixin o botones.$nombre-de-variable.

- Más explícito y claro: Al requerir que uses el namespace, tu código es más claro. Sabes exactamente de dónde viene cada variable o mixin, sin tener que adivinar.

- No duplica código: @use carga cada archivo solo una vez. @import podía cargar el mismo archivo varias veces si se importaba desde diferentes lugares, lo que a veces generaba CSS duplicado.

## Cómo se usa @use

Aquí tienes un ejemplo práctico de cómo se vería una estructura de proyecto con el sistema de módulos:

\_variables.scss

```scss
$color-primario: #007bff;
$font-stack: "Arial", sans-serif;
```

\_botones.scss

```scss
@use "variables" as var; // Le damos un alias 'var' al namespace 'variables'

.boton {
  font-family: var.$font-stack;
  background-color: var.$color-primario;
  color: white;
  padding: 10px 20px;
}
```

styles.scss (Archivo principal)

```scss
@use "variables" as *; // El `as *` es opcional y sirve para importar el contenido sin namespace. Esto es útil para variables globales o mixins de utilidades.
@use "botones"; // Importamos el módulo de botones

body {
  font-family: $font-stack;
  color: #333;
}

.mi-boton {
  @extend botones.boton; // Extendemos la clase base del módulo de botones
}
```

## Resumen del sistema de módulos

- Se usa la directiva `@use 'nombre-del-archivo'`;.

- Se accede al contenido con la sintaxis `namespace.nombre-del-recurso`.

- Puedes usar `as alias`; para acortar el namespace.

- Puedes usar `as *`; si quieres que el contenido se importe sin namespace (lo más común para variables globales).
