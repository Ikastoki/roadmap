<h1 align='center'>Dataset</h1>

<h2>📑 Contenido</h2>

- [Dataset](#dataset)
- [¿Qué es el atributo `data-*` en HTML?](#qué-es-el-atributo-data--en-html)
- [Accediendo al dataset en JavaScript](#accediendo-al-dataset-en-javascript)
- [Modificar atributos `data-*` con dataset](#modificar-atributos-data--con-dataset)
- [Reglas de conversión de `data-*` a dataset](#reglas-de-conversión-de-data--a-dataset)
- [Uso práctico del dataset en aplicaciones web](#uso-práctico-del-dataset-en-aplicaciones-web)
  - [Enriquecimiento de contenido dinámico](#enriquecimiento-de-contenido-dinámico)
  - [Interactividad sin manipular el HTML visible](#interactividad-sin-manipular-el-html-visible)
  - [Gestión de formularios dinámicos](#gestión-de-formularios-dinámicos)
  - [Buenas prácticas al usar `data-*` y `dataset`](#buenas-prácticas-al-usar-data--y-dataset)

## Dataset

El atributo dataset en HTML y JavaScript permite almacenar y acceder a datos personalizados (información adicional) directamente en los elementos HTML. Estos datos se definen en el HTML utilizando atributos `data-*` y luego se pueden leer o manipular fácilmente desde JavaScript a través de la propiedad `dataset`. El uso de `data-*` es una forma estandarizada de incrustar información no visual en los elementos, sin alterar su estructura o presentación.

## ¿Qué es el atributo `data-*` en HTML?

El estándar HTML5 introdujo los atributos `data-*`, que permiten incrustar datos personalizados en cualquier elemento HTML. El formato es simple: los atributos personalizados comienzan con `data-` seguido por un nombre, como en el siguiente ejemplo:

```html
<div id="producto1" data-nombre="Camisa" data-precio="29.99"></div>
```

En este caso, los atributos data-nombre y data-precio almacenan información personalizada sobre el producto en el elemento div. Estos atributos no afectan la apariencia o el comportamiento del div de forma directa, pero pueden ser utilizados posteriormente por JavaScript para manipular el DOM.

## Accediendo al dataset en JavaScript

Para acceder a los valores almacenados en los atributos `data-*` desde JavaScript, se utiliza la propiedad `dataset`. La propiedad `dataset` es un objeto que contiene todos los atributos `data-*` de un elemento. Los nombres de los atributos se transforman en propiedades de `dataset`, eliminando el prefijo `data-` y utilizando camelCase para el nombre del atributo.

```js
<div id="producto1" data-nombre="Camisa" data-precio="29.99"></div>

<script>
  const producto = document.getElementById('producto1');

  // Acceder a los valores de data-* usando dataset
  console.log(producto.dataset.nombre);  // "Camisa"
  console.log(producto.dataset.precio);  // "29.99"
</script>
```

En este ejemplo:

- El valor del atributo `data-nombre` es accesible como `producto.dataset.nombre`.
- El valor de `data-precio` es accesible como `producto.dataset.precio`.

## Modificar atributos `data-*` con dataset

Además de leer los valores, también es posible modificar los atributos `data-*` usando la propiedad dataset en JavaScript. Simplemente asigna un nuevo valor a la propiedad correspondiente del dataset.

```html
<div id="producto1" data-nombre="Camisa" data-precio="29.99"></div>

<script>
  const producto = document.getElementById("producto1");

  // Modificar el valor de data-nombre
  producto.dataset.nombre = "Camiseta";

  // Ver el nuevo valor
  console.log(producto.dataset.nombre); // "Camiseta"
</script>
```

En este ejemplo, el valor del atributo data-nombre se modifica de "Camisa" a "Camiseta" directamente a través del objeto `dataset`.

## Reglas de conversión de `data-*` a dataset

Cuando se trabaja con dataset, los nombres de los atributos `data-*` siguen las siguientes reglas:

El nombre del atributo después de `data-` se convierte en camelCase. Por ejemplo:

- `data-user-id` se convierte en dataset.userId.
- `data-product-price` se convierte en dataset.productPrice.

Los valores de los atributos `data-*` siempre se tratan como cadenas (`string`). Si necesitas otro tipo de datos (por ejemplo, un número o un booleano), debes realizar la conversión manualmente.

```html
<div id="producto1" data-precio="29.99" data-disponible="true"></div>

<script>
  const producto = document.getElementById("producto1");

  // Convertir los valores a tipos específicos
  const precio = parseFloat(producto.dataset.precio); // 29.99 como número
  const disponible = producto.dataset.disponible === "true"; // true como booleano

  console.log(precio); // 29.99
  console.log(disponible); // true
</script>
```

## Uso práctico del dataset en aplicaciones web

El atributo `data-*` y la propiedad `dataset` son especialmente útiles en situaciones donde se necesita almacenar información contextual o datos relacionados con un elemento HTML, pero sin alterar su apariencia visual ni su comportamiento inherente. Estos son algunos casos de uso comunes:

### Enriquecimiento de contenido dinámico

Los atributos `data-*` permiten asociar datos contextuales con elementos, lo que facilita su manipulación desde JavaScript. Por ejemplo, en una lista de productos generada dinámicamente, cada elemento podría llevar atributos `data-*` con el ID del producto, el precio o la categoría.

```js
<ul>
  <li data-id="101" data-nombre="Laptop">Laptop</li>
  <li data-id="102" data-nombre="Teléfono">Teléfono</li>
</ul>

<script>
  const productos = document.querySelectorAll('li');

  productos.forEach(producto => {
    console.log(`ID: ${producto.dataset.id}, Nombre: ${producto.dataset.nombre}`);
  });
</script>
```

### Interactividad sin manipular el HTML visible

Puedes usar `data-*` para agregar interactividad a elementos sin modificar el contenido HTML visible. Por ejemplo, puedes almacenar información adicional sobre botones que desencadenen ciertas acciones, como abrir un modal o hacer una solicitud AJAX.

```html
<button data-user-id="1234" data-action="eliminar">Eliminar usuario</button>

<script>
  const button = document.querySelector("button");

  button.addEventListener("click", () => {
    const userId = button.dataset.userId;
    const action = button.dataset.action;
    console.log(`Acción: ${action} para el usuario ID: ${userId}`);
  });
</script>
```

### Gestión de formularios dinámicos

En formularios dinámicos, los atributos `data-*` son útiles para vincular información adicional a campos de formulario, como identificadores o datos previos para validaciones o procesamiento.

### Buenas prácticas al usar `data-*` y `dataset`

- **No abuses de `data-*`:** Aunque son útiles para almacenar información contextual, evita sobrecargar los elementos HTML con demasiados atributos `data-*`. Si los datos son complejos o grandes, considera almacenar los datos en otro lugar y usar los identificadores para enlazar la información.

- **Mantén los nombres claros y semánticos:** Usa nombres de atributos que describan claramente el propósito de los datos almacenados. Esto facilita la comprensión del código tanto para ti como para otros desarrolladores.

- **Evita usar `data-*` para datos críticos:** Recuerda que los valores de los atributos `data-*` son accesibles para el usuario, lo que significa que pueden ser manipulados. No uses `data-*` para almacenar información sensible, como credenciales o tokens de autenticación.

- **Usa `data-*` en lugar de clases o IDs para almacenar datos:** Aunque es posible usar clases o IDs para almacenar datos personalizados, esta práctica puede dificultar el mantenimiento y la legibilidad del código. El uso de `data-*` es la manera recomendada para almacenar datos adicionales en los elementos HTML.
