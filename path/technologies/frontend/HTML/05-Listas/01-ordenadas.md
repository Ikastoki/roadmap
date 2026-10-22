<h1 align="center">Listas Ordenadas</h1>

<h2>📑 Contenido</h2>

- [Listas Ordenadas](#listas-ordenadas)
  - [Cambiar el número de inicio](#cambiar-el-número-de-inicio)
  - [Cambiar el valor concreto](#cambiar-el-valor-concreto)
  - [Invertir orden](#invertir-orden)
  - [Cambiar tipo](#cambiar-tipo)

## Listas Ordenadas

Para mostrar listas ordenadas en HTML se utiliza las etiquetas: <br>
`<ol> <li>Elementos<li> </ol>`

```HTML
Ejemplo:
    <ol>
      <li>Elemento-1</li>
      <li>Elemento-2</li>
      <li>Elemento-3</li>
    </ol>

Resultado:
    1.Elemento-1
    2.Elemento-2
    3.Elemento-3
```

### Cambiar el número de inicio

Se puede cambiar el número de inicio de las listas ordenadas con el atributo `start="numero"`

```HTML
Ejemplo:
    <ol start="4">
      <li>Elemento-1</li>
      <li>Elemento-2</li>
      <li>Elemento-3</li>
    </ol>

Resultado:
    4.Elemento-1
    5.Elemento-2
    6.Elemento-3
```

### Cambiar el valor concreto

Se puede cambiar el valor concreto de un número en las listas ordenadas con el atributo `value="numero"`

**Importante:** Cambia los números que van después del valor cambiado.

```HTML
Ejemplo:
    <ol>
      <li>Elemento-1</li>
      <li value="44">Elemento-2</li>
      <li>Elemento-3</li>
      <li>Elemento-4</li>
      <li>Elemento-5</li>
    </ol>

Resultado:
    1.Elemento-1
    44.Elemento-2
    45.Elemento-3
    46.Elemento-4
    47.Elemento-5
```

### Invertir orden

Para invertir el orden de las listas(No cambia el orden del contenido) `<ol reversed>`

```HTML
Ejemplo:
    <ol reversed>
      <li>Elemento-1</li>
      <li>Elemento-2</li>
      <li>Elemento-3</li>
    </ol>

Resultado:
    3.Elemento-1
    2.Elemento-2
    1.Elemento-3
```

### Cambiar tipo

Para cambiar el tipo de orden de una lita ordenada:

- Lista decimales(Por defecto).
- Listas alfabéticas en minúsculas-mayúsculas(a-A).
- Listas de números romanos en minúsculas-mayúsculas(i-I).
- Mediante CSS se puede cambiar más tipos: `list-style-type`
- Mejor asignar el tipo de lista mediante CSS.

```HTML
Ejemplo:
    <ol type="a">
      <li>Elemento-1</li>
      <li>Elemento-2</li>
      <li>Elemento-3</li>
    </ol>

Resultado:
    a.Elemento-1
    b.Elemento-2
    c.Elemento-3
```
