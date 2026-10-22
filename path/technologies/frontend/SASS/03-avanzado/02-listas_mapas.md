<h1 align="center">Listas y Mapas</h1>

<h2>📑 Contenido</h2>

- [Listas y Mapas](#listas-y-mapas)
- [Listas](#listas)
- [Mapas](#mapas)
- [Resumen](#resumen)

## Listas y Mapas

Sass te permite almacenar y manipular colecciones de datos, lo que te da una flexibilidad enorme para automatizar y organizar tus estilos.

## Listas

Una lista es una colección de valores separados por comas o espacios. Son muy similares a los arrays de JavaScript. Puedes usar listas para agrupar valores relacionados, como fuentes, colores o tamaños de padding.

```scss
// Lista de colores (separados por comas)
$colores-secundarios: #e9ecef, #adb5bd, #6c757d;

// Lista de fuentes (separados por espacios)
$fuentes-de-reserva: Helvetica, Arial, sans-serif;
```

Funciones útiles para listas:

- `length($lista)`: Devuelve el número de elementos de la lista.

- `nth($lista, $indice)`: Devuelve el elemento en una posición específica (el índice comienza en 1).

- `join($lista1, $lista2)`: Combina dos listas en una.

- `append($lista, $valor)`: Añade un valor al final de una lista.

Ejemplo de uso:

```scss
// Sass
$lista-de-padding: 10px, 20px, 30px;

.elemento {
  padding: nth($lista-de-padding, 2); // Devuelve 20px
}
```

Las listas se usan comúnmente con el bucle `@each` para generar estilos dinámicamente.

## Mapas

Un mapa es una colección de pares clave-valor, similar a un objeto en JavaScript o un diccionario en Python. Cada par clave-valor está separado por comas, y la clave y el valor están separados por dos puntos (`:`). Los mapas son perfectos para almacenar datos de configuración, como los breakpoints o una paleta de colores.

```scss
// Sass
$temas: (
  claro: (
    fondo: #fff,
    texto: #333,
  ),
  oscuro: (
    fondo: #333,
    texto: #fff,
  ),
);

// Mapa de breakpoints que vimos antes
$breakpoints: (
  "movil": 576px,
  "tablet": 768px,
);
```

Funciones útiles para mapas:

- `map-get($mapa, $clave)`: Devuelve el valor asociado a una clave específica.

- `map-has-key($mapa, $clave)`: Comprueba si existe una clave en el mapa.

- `map-merge($mapa1, $mapa2)`: Combina dos mapas.

```scss
// Sass
@function obtener-color($tema, $tipo) {
  @return map-get(map-get($temas, $tema), $tipo);
}

body {
  background-color: obtener-color("claro", "fondo");
  color: obtener-color("claro", "texto");
}

@media (min-width: map-get($breakpoints, "tablet")) {
  // Estilos responsivos...
}
```

## Resumen

- Usa listas para agrupar valores simples.

- Usa mapas para almacenar datos de configuración o pares clave-valor.
