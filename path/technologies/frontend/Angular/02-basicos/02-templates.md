<h1 align="center">Templates</h1>

<h2>📑 Contenido</h2>

- [Templates](#templates)
- [Interpolación ({{ }})](#interpolación--)
- [Enlace de Propiedades (\[ \])](#enlace-de-propiedades--)
- [Enlace de Eventos (( ))](#enlace-de-eventos--)
- [Enlace Bidireccional (\[( )\])](#enlace-bidireccional--)

## Templates

Las plantillas, o templates, son la parte HTML de un componente. Aquí es donde se define la vista de tu aplicación. Para hacer que la vista sea dinámica, Angular utiliza una sintaxis especial para el enlace de datos (data binding), que permite la comunicación entre el template y la clase del componente.

## Interpolación ({{ }})

La interpolación es la forma más básica de mostrar datos de tu componente en el template. Utiliza la sintaxis de doble llave para insertar el valor de una propiedad de la clase TypeScript directamente en el HTML.

```html
<p>Hola, mi nombre es **{{ nombre }}**.</p>
```

En este caso, Angular sustituye {{ nombre }} por el valor de la propiedad nombre definida en la clase del componente.

## Enlace de Propiedades ([ ])

El enlace de propiedades te permite asignar un valor de una propiedad del componente a una propiedad de un elemento del DOM. Es útil para pasar valores que no son solo texto, como atributos booleanos, valores numéricos o estilos.

```html
<button [disabled]="esBotonDeshabilitado">Haz clic</button>
```

Si la propiedad esBotonDeshabilitado en tu componente es true, el botón se deshabilitará.

## Enlace de Eventos (( ))

El enlace de eventos te permite escuchar eventos del usuario en el template (como `click`, `mouseover`, `submit`) y ejecutar un método de la clase del componente en respuesta.

```html
<button (click)="manejarClick()">Click me</button>
```

Cuando el usuario hace clic en el botón, el método `manejarClick()` en el componente se ejecuta.

## Enlace Bidireccional ([( )])

El enlace bidireccional o two-way binding combina el enlace de propiedades y el de eventos. Permite un flujo de datos en dos direcciones: si el usuario cambia el valor en el template, la propiedad del componente se actualiza automáticamente, y si la propiedad del componente cambia, el valor en el template se actualiza.

```html
<input [(ngModel)]="nombreUsuario" />
```

El valor del input se sincroniza con la propiedad nombreUsuario. Para usar `[(ngModel)]`, debes importar el FormsModule en tu módulo.
