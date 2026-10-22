<h1 align="center">Líneas por nombre</h1>

<h2>📑 Contenido</h2>

- [Líneas por nombre](#líneas-por-nombre)
  - [Ejemplo: Layout(Header,Aside,Content,Footer)](#ejemplo-layoutheaderasidecontentfooter)

## Líneas por nombre

Cuando se crean las líneas automáticamente se les asigna un numero, estos números establecen donde empiezan(Numero positivo) y terminan las lineas(Numero negativo). Sabiendo donde empiezan y terminan las líneas se pueden establecer donde empiezan y terminan.
En las propiedades `grid-template-columns` y `grid-template-rows` asignaremos los nombres y el espacio que ocuparan. Y en los items asignamos donde comienzan y lo que se expanden `span`.

### Ejemplo: Layout(Header,Aside,Content,Footer)

```html
<!-- HTML -->
<div class="wrapper">
  <header class="flex">Header</header>
  <aside class="flex">Aside</aside>
  <article class="flex">Content</article>
  <footer class="flex">Footer</footer>
</div>
```

```css
/* CSS */

.wrapper {
  height: 100vh;
  display: grid;
  gap: 5px; /* Igual que flex, separa los items*/
  grid-template-columns:
    [aside-start] 1fr
    [article-start] 2fr;

  grid-template-rows:
    [header-start] 1fr
    [article-start] 2fr
    [footer-start] 1fr;
}

header {
  background-color: #68a7ad;
  grid-column: aside-start / span 2;
}
aside {
  background-color: #eee4ab;
}
article {
  background-color: #99c4c8;
}
footer {
  background-color: #e5cb9f;
  grid-column: aside-start / span 2;
}
/* Centrar los textos */
.flex {
  display: flex;
  justify-content: center;
  align-items: center;
}
```

![Grid líneas por nombre](./img/nombre-lineas.png)
