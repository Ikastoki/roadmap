<h1 align="center">Directivas</h1>

<h2>📑 Contenido</h2>

- [Directivas](#directivas)
- [Directivas Estructurales](#directivas-estructurales)
- [Métodos antiguos (\*ngIf, \*ngFor)](#métodos-antiguos-ngif-ngfor)
- [Métodos nuevos (a partir de Angular v17) (@if, @for, @switch)](#métodos-nuevos-a-partir-de-angular-v17-if-for-switch)
- [Directivas de Atributo](#directivas-de-atributo)

## Directivas

Las directivas son clases de TypeScript que modifican el comportamiento o la apariencia de los elementos del DOM. Angular las utiliza para añadir una lógica especial a las plantillas (templates) que va más allá del simple HTML.

## Directivas Estructurales

Estas directivas tienen el poder de modificar la estructura del DOM añadiendo, eliminando o manipulando elementos. Su sintaxis siempre comienza con un asterisco (`*`).

## Métodos antiguos (*ngIf, *ngFor)

- `*ngIf`: Permite renderizar condicionalmente un bloque de HTML. El elemento se añade al DOM si la condición es true y se elimina si es false.

```html
<p *ngIf="mostrarMensaje">Este mensaje es visible.</p>
```

- `*ngFor`: Se utiliza para iterar sobre una colección de datos (como un array) y renderizar un template para cada elemento.

```html
<ul>
  <li *ngFor="let usuario of usuarios">{{ usuario.nombre }}</li>
</ul>
```

## Métodos nuevos (a partir de Angular v17) (@if, @for, @switch)

Angular introdujo una nueva sintaxis de flujo de control (`control flow`) que es más eficiente y fácil de leer. Estas directivas no usan asterisco.

- `@if`: El equivalente moderno de `*ngIf`, con una sintaxis más clara y con bloques opcionales else para manejar ambos escenarios.

```html
@if (mostrarMensaje) {
<p>Este mensaje es visible.</p>
} @else {
<p>Este mensaje no es visible.</p>
}
```

- `@for`: El equivalente moderno de `*ngFor`, que incluye bloques track, empty y separator opcionales.

```html
<ul>
  @for (usuario of usuarios; track usuario.id) {
  <li>{{ usuario.nombre }}</li>
  } @empty {
  <li>No hay usuarios disponibles.</li>
  }
</ul>
```

`@switch`: Permite renderizar condicionalmente diferentes bloques de HTML según un valor, reemplazando la directiva `*ngSwitch`.

```html
@switch (estado) { @case ('activo') {
<p>Usuario activo</p>
} @case ('inactivo') {
<p>Usuario inactivo</p>
} @default {
<p>Estado desconocido</p>
} }
```

## Directivas de Atributo

Estas directivas no cambian la estructura del DOM, sino que modifican la apariencia o el comportamiento de un elemento.

- `ngClass`: Permite añadir o eliminar clases CSS de un elemento condicionalmente.

  ```html
  <button [ngClass]="{'activo': estaActivo}">Estado</button>
  ```

- `ngStyle`: Permite aplicar estilos CSS inline a un elemento condicionalmente.

  ```html
  <p [ngStyle]="{'color': estaActivo ? 'green' : 'red'}">Estado de la cuenta</p>
  ```

- Directivas de atributo propias: Puedes crear tus propias directivas de atributo para añadir lógica reutilizable a cualquier elemento. Por ejemplo, una directiva para resaltar un elemento al pasar el ratón. Para crearlas, se usa el comando `ng generate directive nombre-directiva` y el decorador `@Directive`.
