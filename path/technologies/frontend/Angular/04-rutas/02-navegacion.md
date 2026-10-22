<h1 align="center">Navegación</h1>

<h2>📑 Contenido</h2>

- [Router outlet](#router-outlet)
- [Directiva routerLink](#directiva-routerlink)
  - [Uso simple: Para navegar a una ruta estática.](#uso-simple-para-navegar-a-una-ruta-estática)
  - [Uso con array: Es el método más flexible, ideal para pasar parámetros a la ruta.](#uso-con-array-es-el-método-más-flexible-ideal-para-pasar-parámetros-a-la-ruta)
- [Servicio Router](#servicio-router)

## Router outlet

El `<router-outlet>` es una directiva especial que funciona como un marcador de posición. Cuando el usuario navega a una ruta específica, Angular inyecta dinámicamente el componente correspondiente a esa ruta en el lugar donde se encuentra el `<router-outlet>`.

Normalmente, este elemento se coloca en el componente principal de tu aplicación, el AppComponent, para que todas las vistas de la aplicación se muestren en ese lugar.

```html
<nav></nav>

<router-outlet></router-outlet>
```

## Directiva routerLink

El `routerLink` es una directiva de atributo que se utiliza para crear enlaces de navegación. Es la forma declarativa de navegar en Angular y es el equivalente a un `href` en HTML, pero sin recargar la página.

### Uso simple: Para navegar a una ruta estática.

```html
<a routerLink="/productos">Ver productos</a>
```

### Uso con array: Es el método más flexible, ideal para pasar parámetros a la ruta.

```html
<a [routerLink]="['/productos', '123']">Ver producto 123</a>
```

## Servicio Router

El servicio `Router` es una clase inyectable que te permite controlar la navegación de forma programática desde la lógica de tu componente. Es ideal para situaciones donde la navegación no es un simple clic, como después de enviar un formulario, en una redirección condicional, o cuando se maneja un evento complejo.

Para usarlo, primero debes inyectarlo en el constructor del componente:

```typescript
import { Router } from "@angular/router";

export class MiComponente {
  constructor(private router: Router) {}

  // Un método para navegar después de una acción
  guardarYRedirigir() {
    // ... lógica para guardar datos ...

    // Navega de forma programática a la ruta '/dashboard'
    this.router.navigate(["/dashboard"]);
  }
}
```

La sintaxis del método `Maps` es similar al uso de `routerLink` con un array, lo que facilita la navegación a rutas más complejas con parámetros.
