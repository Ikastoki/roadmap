<h1 align="center">Medios y Condiciones</h1>

<h2>📑 Contenido</h2>

- [@media](#media)
- [@supports](#supports)
- [@scope](#scope)

## @media

Se utiliza para aplicar estilos específicos según las características del dispositivo o la ventana gráfica del navegador. Permite crear diseños responsive adaptados a diferentes tamaños de pantalla, resoluciones o características del dispositivo.

```css
/* CSS */
@media tipo_de_medio and (condición) {
  /* Estilos a aplicar si la condición se cumple */
}
```

> [!NOTE]
>
> Mirar más en [Media Queries](../03-Layouts/01-Basicos/07-media_query.md)

## @supports

Se utiliza para verificar si el navegador es compatible con ciertas características o propiedades antes de aplicar un conjunto específico de estilos. Esto permite que los desarrolladores adapten los estilos según la capacidad del navegador.

**Condiciones:**

- `@supports (condición)` Aplica los estilos si se cumple la condición.
- `@supports not (condición)` Aplica los estilos si no se cumple la condición.
- `@supports (condición1) and (condición2)` Aplica los estilos si se cumplen las diferentes condiciones.
- `@supports (condición1) or (condición2)` Aplica los estilos si se cumple una de las dos condiciones.

```html
<!-- HTML -->
<nav>
  <ul>
    <li><a href="#">Inicio</a></li>
    <li><a href="#">Acerca de</a></li>
    <li><a href="#">Servicios</a></li>
    <li><a href="#">Contacto</a></li>
  </ul>
</nav>
```

```css
/* CSS */
ul {
  display: flex;
  gap: 20px;
  list-style: none;
}

@supports not (display: flex) {
  li {
    list-style: none;
    float: left;
    padding: 20px;
  }
}
```

## @scope

Permite seleccionar elementos en subárboles DOM específicos, apuntando a elementos con precisión sin escribir selectores demasiado específicos que sean difíciles de anular y sin acoplar demasiado sus selectores a la estructura DOM.

Aplica estilo a un scope concreto.

**Sintaxis básica**

```html
<!-- HTML -->
<div class="ambito">
  <h2 class="encabezados">Encabezado en el ambito.</h2>
</div>
<h2 class="encabezados">Encabezado fuera del ambito.</h2>
```

```css
/* CSS */
@scope (.ambito) {
  .encabezados {
    background: teal;
    color: greenyellow;
    padding: 15px;
  }
}
```

**Podemos marcar un principio y un final de ámbito en nuestro DOM**

```css
/* CSS */
@scope (límite inicial) to (límite final) {
  /* Estilos CSS */
}
```
