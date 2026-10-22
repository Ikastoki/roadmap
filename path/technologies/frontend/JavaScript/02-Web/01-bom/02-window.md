<h1 align='center'>El Objeto Window</h1>

<h2>📑 Contenido</h2>

- [El Objeto Window](#el-objeto-window)
- [Window](#window)
- [Propiedades clave del objeto window:](#propiedades-clave-del-objeto-window)
- [El objeto window como objeto global](#el-objeto-window-como-objeto-global)

## El Objeto Window

El objeto principal del BOM es `window`, que representa la ventana del navegador o el marco donde se ejecuta la página web. A través de `window`, JavaScript puede acceder a otras características del navegador como la barra de direcciones, historial, tamaño de la ventana y más.

El objeto `window` es el objeto global en el navegador, lo que significa que cualquier función o variable definida en el ámbito global es parte de él. Por ejemplo, cuando defines una variable global en JavaScript, realmente estás creando una propiedad del objeto `window`.

## Window

El objeto `window` es el objeto raíz del BOM y proporciona acceso a muchas propiedades y métodos útiles para interactuar con la ventana del navegador. Aunque algunas de sus funcionalidades se solapan con las de otros objetos (como `document` para el DOM), el objeto `window` maneja principalmente las operaciones que afectan a la propia ventana o el entorno del navegador.

## Propiedades clave del objeto window:

- **window.document:** Representa el objeto document, que permite interactuar con el DOM (árbol de elementos HTML).
  window.innerHeight y window.innerWidth: Devuelven la altura y el ancho de la ventana del contenido visible.
- **window.location:** Proporciona acceso a la URL actual y permite cambiarla o redirigir al usuario a una nueva ubicación.
  window.history: Permite acceder al historial de navegación del usuario y moverse entre páginas visitadas.
- **window.navigator:** Ofrece información sobre el navegador, como el nombre, la versión y el sistema operativo.
  window.screen: Proporciona información sobre la pantalla del dispositivo, como la resolución.
- **window.localStorage y window.sessionStorage:** Permiten almacenar datos en el navegador de manera persistente (localStorage) o temporal (sessionStorage).
- **window.alert(), window.confirm(), window.prompt():** Métodos para mostrar cuadros de diálogo que interactúan con el usuario.

## El objeto window como objeto global

El objeto `window` es el objeto global por defecto en los navegadores. Esto significa que todas las variables y funciones declaradas en el ámbito global (fuera de cualquier función) se convierten en propiedades de `window`.

```js
var nombre = "Juan";
console.log(window.nombre); // Imprime "Juan"
```

En este ejemplo, la variable nombre se convierte en una propiedad de ``window``, por lo que puede ser accedida como ``window.nombre``.
