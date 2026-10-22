<h1 align="center">Listas No Ordenadas (Desordenadas)</h1>

<h2>📑 Contenido</h2>

- [Listas No Ordenadas(Desordenadas)](#listas-no-ordenadasdesordenadas)
  - [Cambiar símbolo](#cambiar-símbolo)

## Listas No Ordenadas(Desordenadas)

Para mostrar listas no ordenadas en HTML se utiliza las etiquetas:<br>
`<ul> <li>Elementos<li> </ul>`

```HTML
Ejemplo:
    <ul>
      <li>Elemento-1</li>
      <li>Elemento-2</li>
      <li>Elemento-3</li>
    </ul>

Resultado:
    • Elemento-1
    • Elemento-2
    • Elemento-3
```

### Cambiar símbolo

Se puede cambiar el símbolo de la lista desordenada:
`<ul type="square">`

- Por defecto las listas desordenas usan el símbolo de •
- Los 4 tipos mas comunes son:
  - **dist:** Para poner el símbolo por defecto.
  - **circle:** Para poner un símbolo circular.
  - **square:** Para poner un símbolo cuadrado.
  - **none:** Para eliminar la marca.
- Mejor asignar el tipo de lista mediante CSS.

```HTML
Ejemplo:
    <ul type="circle">
      <li>Elemento-1</li>
      <li>Elemento-2</li>
      <li>Elemento-3</li>
    </ul>

Resultado:
    ◌ Elemento-1
    ◌ Elemento-2
    ◌ Elemento-3
```

