<h1 align="center">Map</h1>

<h2>📑 Contenido</h2>

- [Map](#map)
- [Cords](#cords)
  - [Rectangular (shape="rect")](#rectangular-shaperect)
  - [Circular (shape="circle")](#circular-shapecircle)
  - [Poligonal (shape="poly")](#poligonal-shapepoly)

## Map

La etiqueta `<map>`(mapa de imágenes) se utiliza junto con las etiquetas `<area>` para crear mapas de imagen interactivos. Un mapa de imagen permite dividir una imagen en áreas o regiones clicables, cada una asociada a un enlace o acción diferente. Puedes utilizar la etiqueta `<map>` para definir las áreas y luego vincularlas a URLs específicas o activar funciones JavaScript cuando el usuario interactúa con esas áreas.

```html
<!-- HTML -->
<img src="continentes.jpg" usemap="#mapa-continentes" />

<map name="mapa-continentes">
  <area
    target="_blank"
    alt="Africa"
    title="Africa"
    href="https://es.wikipedia.org/wiki/%C3%81frica"
    coords="395,111,215,258"
    shape="rect"
  />
</map>
```

> [!IMPORTANT]
>
> La etiqueta `<map>` debe tener un atributo name que coincide con el atributo usemap de la etiqueta de imagen para establecer la asociación entre la imagen y el mapa.

## Cords

El atributo coords en las etiquetas `<area>` de un mapa de imagen HTML se utiliza para especificar las coordenadas de una forma geométrica que define la región de la imagen que es sensible al clic del usuario. La forma puede ser rectangular, circular o poligonal, dependiendo del valor del atributo shape. Las coordenadas deben proporcionarse en píxeles.

El valor exacto del atributo coords varía según la forma especificada. Aquí hay una breve descripción de cómo se definen las coordenadas para diferentes formas:

### Rectangular (shape="rect")

- El atributo coords tiene cuatro valores: left, top, right, y bottom.
- Ejemplo: coords="10,20,50,40" indica una región rectangular con esquinas en (10, 20) y (50, 40).

### Circular (shape="circle")

- El atributo coords tiene tres valores: x, y, y radio.
- Ejemplo: coords="50,50,30" indica una región circular con centro en (50, 50) y radio de 30 píxeles.

### Poligonal (shape="poly")

- El atributo coords tiene una lista de pares x,y que representan los vértices del polígono.
- Ejemplo: coords="10,10,30,40,50,20" indica un triángulo con vértices en (10,10), (30,40), y (50,20).

> [!TIP]
>
> Usar generadores de map como [Image Map Generator](https://www.image-map.net/).


