<h1 align="center">BEM(Block Element Modifier)</h1>

<h2>📑 Contenido</h2>

- [BEM](#bem)
  - [Bloque](#bloque)
  - [Elemento](#elemento)
  - [Modificadores](#modificadores)

## BEM

La metodología BEM(Block Element Modifier) es un sistema de nomenclaturas de clases. Se utiliza para diferenciar clases que cumplen diferentes funciones. Para ello se nombran las clases de tal forma que sea el nombre el que describa la función de la clase. BEM se aplica por bloques, se usa dos guiones bajos para el bloque de un elemento y dos guiones medios para separar el modificador.

- El bloque se separa del elemento con: `bloque__elemento`
- El bloque/elemento se separa del modificador con: `bloque__elemento--modificador`
- Palabras compuestas se separan con: `palabra-compuesta`

### Bloque

**Bloque\_\_**
El bloque tiene que tener un significado por si solo: header,menu,tabs,footer...

### Elemento

**\_\_Elemento**
Los elementos son las partes del bloque que no tienen un significado independiente. Están de alguna forma vinculados al bloque. Un titulo o una descripción de un bloque `card`, un logo o una lista de un `nav`.

### Modificadores

**--Modificadores**
Los modificadores se usan para cambiar la apariencia o el comportamiento del elemento o el bloque.
Usaremos los modificadores para cambiar un color, un tamaño, si esta habilitado o deshabilitad...

Ejemplo:

```html
<!-- HTML -->
<nav class="navegador-principal">
  <ul class="navegador-principal__lista">
    <li class="navegador-principal__item">item1</li>
    <li class="navegador-principal__item">item2</li>
    <li class="navegador-principal__item">
      <a class="navegador-principal__enlace" href="#">item3</a>
    </li>
    <li class="navegador-principal__item">
      <a class="navegador-principal__enlace--rojo" href="#">item4</a>
    </li>
  </ul>
</nav>
```

> [!NOTE]
>
> Recomendable visitar los "cheat sheet" de [9elements](https://9elements.com/bem-cheat-sheet/#links), en esta web se aprecia muy bien las partes del bloque,elemento,modificador.
