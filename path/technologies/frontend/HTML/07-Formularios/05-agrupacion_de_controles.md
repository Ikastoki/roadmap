<h1 align="center">Agrupación de Controles</h1>

<h2>📑 Contenido</h2>

- [Agrupación de Controles](#agrupación-de-controles)
  - [Agrupar Select](#agrupar-select)

## Agrupación de Controles

Para agrupar los campos de un formulario en grupos y tener una mejor organización HTML dispone de la etiqueta `<fieldset>`. Mediante es uso de la etiqueta `<legend>` podemos asignar un titulo a la agrupación del `<fieldset>`.

```html
<!-- Ejemplo -->
<fieldset>
  <legend>Información Personal</legend>
  Nombre: <input name="nombre" type="text" />
  <br />
  <br />
  Apellidos: <input name="apellidos" type="text" />
</fieldset>
```

### Agrupar Select

Para agrupar en diferentes categorías las opciones de un select podemos hacer uso del elemento `<optgroup label="NombreCategoría">`

```html
<!-- Ejemplo -->
<select>
  <option value="">Elija una opción</option>
  <optgroup label="Frontend">
    <option value="">HTML</option>
    <option value="">CSS</option>
    <option value="">JavaScript</option>
  </optgroup>
  <optgroup label="Backend">
    <option value="">Java</option>
    <option value="">PHP</option>
    <option value="">SQL</option>
  </optgroup>
</select>
```


