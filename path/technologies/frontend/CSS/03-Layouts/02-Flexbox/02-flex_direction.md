<h1 align="center">Flex Direction</h1>

<h2>📑 Contenido</h2>

- [Flex Direction](#flex-direction)
  - [Row (por defecto)](#row-por-defecto)
  - [Row Reverse](#row-reverse)
  - [Column](#column)
  - [Column Reverse](#column-reverse)
- [Dirección de Escritura](#dirección-de-escritura)

## Flex Direction

Flex-direction modifica el eje principal.
Es important saber cual es el eje principal para poder mover los items de forma correcta.

- Cuando esta en row o row-reverse el eje principal sera el horizontal, **la fila**.
- Cuando esta en column o column-reverse el eje principal sera el vertical, **la columna**.

> Sintaxis: ` flex-direction: row | row-reverse | column | column-reverse;`

Ejemplos:

**Código**

```HTML
<!-- HTML -->
    <div class="flex-container">
      <div class="flex-item">1 item</div>
      <div class="flex-item">2 item</div>
      <div class="flex-item">3 item</div>
      <div class="flex-item">4 item</div>
      <div class="flex-item">5 item</div>
    </div>
```

```CSS
/* CSS */
   .flex-container {
        width: 400px;
        height: 300px;
        margin: 15% auto;
        display: flex;
        gap: 5px;
        flex-direction: row;
        background-color: #937dc2;
      }

      .flex-item {
        width: 50px;
        height: 50px;
        padding: 3px;
        font-size: 20px;
      }
      .flex-item:nth-child(even) {
        background-color: #9fc9f3;
      }

      .flex-item:nth-child(odd) {
        background-color: #ff9494;
      }
```

### Row (por defecto)

![Flex Direction Row](./img/direction-row.png)

---

### Row Reverse

![Flex Direction Row Reverse](./img/direction-row-reverse.png)

---

### Column

![Flex Direction Column](./img/direction-column.png)

---

### Column Reverse

![Flex Direction Row Reverse](./img/direction-column-reverse.png)

## Dirección de Escritura

A la hora de utilizar flex es importante tener en cuenta la dirección de escritura. Si utilizamos la propiedad `direction: rtl` afectará la dirección en la que se colocan los elementos flexibles dentro de un contenedor con `display: flex`. Al establecer `direction: rtl` (derecha a izquierda), cambiará la dirección de escritura de los elementos flexibles y, por lo tanto, también influirá en su disposición dentro del contenedor.

Ejemplo:

```html
<!-- HTML -->
<div class="flex-container">
  <div class="flex-item">1 item</div>
  <div class="flex-item">2 item</div>
  <div class="flex-item">3 item</div>
  <div class="flex-item">4 item</div>
  <div class="flex-item">5 item</div>
</div>
```

```css
/* CSS */
.flex-container {
  width: 400px;
  height: 300px;
  margin: 15% auto;
  display: flex;
  gap: 5px;
  flex-direction: row;
  background-color: #937dc2;
  /* Cambiar dirección */
  direction: rtl;
}

.flex-item {
  width: 50px;
  height: 50px;
  padding: 3px;
  font-size: 20px;
}
.flex-item:nth-child(even) {
  background-color: #9fc9f3;
}

.flex-item:nth-child(odd) {
  background-color: #ff9494;
}
```

![Flex Dirección de escritura](./img/direccion-escritura.png)

> [!NOTE]
>
> Lo mismo pasaría con la propiedad `writing-mode: vertical-lr;` <br>
> La dirección pasaría a ser vertical de izquierda a derecha(Arriba-Abajo).
