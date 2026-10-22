<h1 align="center">Configuración de enrutamiento</h1>

<h2>📑 Contenido</h2>

- [Configuración de enrutamiento](#configuración-de-enrutamiento)
- [Módulo de Enrutamiento (AppRoutingModule)](#módulo-de-enrutamiento-approutingmodule)
- [El array Routes](#el-array-routes)
- [Ejemplo de Configuración](#ejemplo-de-configuración)

## Configuración de enrutamiento

La configuración de enrutamiento es el proceso de definir qué componentes de tu aplicación se deben mostrar según la URL en el navegador. Esto es lo que permite que una aplicación de Angular funcione como una Single Page Application (SPA), permitiendo la navegación entre diferentes vistas sin recargar la página completa.

## Módulo de Enrutamiento (AppRoutingModule)

La configuración del enrutamiento se gestiona generalmente en un módulo dedicado llamado `AppRoutingModule`, que se crea automáticamente si habilitas el enrutamiento al generar un nuevo proyecto con el Angular CLI (`ng new mi-app --routing`). Este módulo es donde defines todas las rutas de tu aplicación en un array de objetos de tipo `Routes`.

## El array Routes

El corazón de la configuración es un array de objetos `Routes`. Cada objeto define una única ruta y le dice a Angular qué hacer cuando la URL del navegador coincide con una de ellas. Las propiedades clave de una ruta son:

- `path`: La ruta de la URL que queremos asociar a un componente. Un string vacío `''` representa la ruta raíz de la aplicación.

- `component`: El componente que se debe renderizar cuando se visita esa ruta.

- `redirectTo`: En lugar de renderizar un componente, redirige a otra ruta. Se usa a menudo para la página de inicio o rutas por defecto.

- `pathMatch`: Le dice a Angular cómo debe coincidir la ruta. El valor `full` es el más común y significa que la URL debe coincidir exactamente con el path de la ruta.

## Ejemplo de Configuración

Aquí tienes un ejemplo de cómo se vería un archivo típico `app-routing.module.ts` con algunas rutas

```typescript
import { NgModule } from "@angular/core";
import { RouterModule, Routes } from "@angular/router";

// Importa los componentes que quieres asociar a las rutas
import { HomeComponent } from "./home/home.component";
import { ProductsComponent } from "./products/products.component";
import { ContactComponent } from "./contact/contact.component";
import { NotFoundComponent } from "./not-found/not-found.component";

// Define tus rutas en el array 'routes'
const routes: Routes = [
  // Ruta por defecto que redirige a la página de inicio
  { path: "", redirectTo: "/home", pathMatch: "full" },
  // Ruta para la página de inicio
  { path: "home", component: HomeComponent },
  // Ruta para la página de productos
  { path: "products", component: ProductsComponent },
  // Ruta para la página de contacto
  { path: "contact", component: ContactComponent },
  // Ruta de comodín para cualquier URL que no coincida (404)
  { path: "**", component: NotFoundComponent },
];

@NgModule({
  // Importa y configura las rutas
  imports: [RouterModule.forRoot(routes)],
  // Exporta el módulo de enrutamiento para que esté disponible globalmente
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

En este ejemplo, si la URL del navegador es `/products`, Angular renderizará el `ProductsComponent`. Si la URL no coincide con ninguna ruta definida, se mostrará el `NotFoundComponent` gracias a la ruta comodín `**`.
