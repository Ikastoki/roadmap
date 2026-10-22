<h1 align="center">Control de Opciones</h1>

<h2>📑 Contenido</h2>

- [Control de Opciones](#control-de-opciones)
  - [Checkbox](#checkbox)
  - [Radio](#radio)
  - [Select](#select)

## Control de Opciones

Los elementos de control de opciones permiten a los usuarios seleccionar una o más opciones de una lista de valores predefinidos. Checkbox, Radio y Select.

### Checkbox

El elemento input de tipo `checkbox` permite mediante una caja seleccionar varias opciones. Estos datos se envían en forma de array.

- Se puede hacer uso del atributo `checked` para indicar las opciones.
- Los datos que se envían son el valor que se introduce en el atributo `<name>`.

```html
<!-- Ejemplo -->
<label>Checkbox: (Selecciona tus intereses)</label>

<input type="checkbox" name="peliculas" /> Películas
<input type="checkbox" name="deportes" /> Deportes
<input type="checkbox" name="videojuegos" /> Videojuegos
```

### Radio

El elemento `radio` permite mediante un botón circular seleccionar una única opción si tienen el mismo atributo `name`.

- Atributo `name` asocia el valor del selector a la hora de recoger los datos.
- El valor que se envía es el que se encuentra dentro del `value`.

```html
<!-- Ejemplo -->
<label>Radio: (Selecciona un color)</label>
<br />
<input type="radio" name="color" value="rojo" /> Rojo
<br />
<input type="radio" name="color" value="verde" /> Verde
<br />
<input type="radio" name="color" value="azul" /> Azul
```

### Select

El elemento `select` permite mediante un desplegable elegir una única opción.

- Atributo `name` asocia el valor del selector a la hora de recoger los datos.
- El valor que se envía es el que se encuentra dentro del `value`.

```html
<!-- Ejemplo -->
<select name="provincia">
  <option value="">Elige Provincia</option>
  <option value="Álava/Araba">Álava/Araba</option>
  <option value="Albacete">Albacete</option>
  <option value="Alicante">Alicante</option>
</select>
```


