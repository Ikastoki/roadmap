<h1 align="center">Etiquetados de Control</h1>

<h2>📑 Contenido</h2>

- [Etiquetados de Control](#etiquetados-de-control)

## Etiquetados de Control

Las etiquetas de controles sirven para asociar una descripción corta para los campos.

- El atributo `for` asocia la etiqueta `<label>` con el `id` del campo.
- El `for` y el `id` tienen que coincidir.
- Al hacer click en el `<label>` hace autofocus en el campo.

```HTML
<!-- Ejemplo -->
    <label for="nombre">Nombre</label>
    <input type="text" id="nombre" name="nombre" />
```

> [!TIP]
>
> Otra forma de asociar con label sin **for** ni **id**:
>
> ```html
> <label>
>   Nombre:
>   <input type="text" name="nombre" />
> </label>
> ```


