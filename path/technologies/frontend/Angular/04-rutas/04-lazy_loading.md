<h1 align="center">Lazy Loading</h1>

<h2>📑 Contenido</h2>

- [Lazy Loading](#lazy-loading)
- [¿Por qué es importante el Lazy Loading?](#por-qué-es-importante-el-lazy-loading)
- [Cómo implementar el Lazy Loading](#cómo-implementar-el-lazy-loading)
  - [Crea un módulo para el área de la aplicación](#crea-un-módulo-para-el-área-de-la-aplicación)
  - [Configura las rutas dentro del módulo](#configura-las-rutas-dentro-del-módulo)
  - [Carga previa de módulos (Preloading)](#carga-previa-de-módulos-preloading)

## Lazy Loading

El Lazy Loading (o "carga diferida") es un patrón de diseño en Angular que te permite cargar módulos, componentes y otros recursos de la aplicación solo cuando el usuario los necesita. Esto contrasta con el modelo de carga por defecto, donde todos los módulos se cargan al iniciar la aplicación.

## ¿Por qué es importante el Lazy Loading?

Cuando una aplicación de Angular crece, también lo hace su bundle de JavaScript. Un bundle grande significa que el usuario tardará más en descargar todos los archivos necesarios para ver la página, lo que se traduce en una mala experiencia de usuario y una menor puntuación en métricas de rendimiento como Lighthouse.

El lazy loading resuelve este problema dividiendo el bundle principal en fragmentos más pequeños. El navegador solo descarga el código de un módulo cuando el usuario navega a la ruta que lo utiliza, lo que reduce drásticamente el tiempo de carga inicial de la aplicación.

## Cómo implementar el Lazy Loading

Para implementar la carga diferida, debes configurar tus rutas para que carguen un módulo en lugar de un componente. El proceso es el siguiente:

### Crea un módulo para el área de la aplicación

Imagina que quieres cargar de forma diferida el módulo de "Administración". Primero, debes crear un módulo con su propio archivo de enrutamiento.

Utiliza el Angular CLI para generar un módulo y su enrutamiento asociado con un solo comando:

```bash
ng generate module admin --route admin --module app.module
```

- `ng generate module admin`: Crea el módulo AdminModule.

- `--route admin`: Crea un archivo de enrutamiento (admin-routing.module.ts) y lo configura para la ruta /admin.

- `--module app.module`: Configura el AppModule para que utilice lazy loading para este módulo.

El CLI modificará tu `app-routing.module.ts` para que se vea así:

```typescript
// app-routing.module.ts
import { NgModule } from "@angular/core";
import { Routes, RouterModule } from "@angular/router";

const routes: Routes = [
  // Carga diferida del módulo Admin
  {
    path: "admin",
    loadChildren: () =>
      import("./admin/admin.module").then((m) => m.AdminModule),
  },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

La clave aquí es la propiedad `loadChildren`, que le dice al enrutador que, en lugar de cargar un componente directamente, debe cargar el módulo `AdminModule` de forma diferida.

### Configura las rutas dentro del módulo

Una vez que el módulo AdminModule se ha cargado, el enrutador necesita saber qué componentes mostrar. Esto se configura en su propio archivo de enrutamiento (admin-routing.module.ts).

```typescript
// admin-routing.module.ts
import { NgModule } from "@angular/core";
import { Routes, RouterModule } from "@angular/router";
import { DashboardComponent } from "./dashboard/dashboard.component"; // Tu componente

const routes: Routes = [
  { path: "", component: DashboardComponent }, // La ruta vacía (o 'admin')
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule],
})
export class AdminRoutingModule {}
```

Es muy importante notar que en este módulo se utiliza `RouterModule.forChild()` en lugar de `forRoot()`. Esto le dice a Angular que estas son rutas secundarias de un módulo que se cargará de forma diferida.

### Carga previa de módulos (Preloading)

A veces, quieres lo mejor de ambos mundos: la carga rápida inicial y que el resto de los módulos se descarguen en segundo plano mientras el usuario interactúa con la aplicación. Para ello, puedes configurar una estrategia de preloading.

Simplemente modifica el forRoot en `app-routing.module.ts` de esta manera:

```typescript
import { PreloadAllModules, RouterModule, Routes } from "@angular/router";

@NgModule({
  imports: [
    RouterModule.forRoot(routes, { preloadingStrategy: PreloadAllModules }),
  ],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

El `PreloadAllModules` es una estrategia de precarga incluida en Angular que carga todos los módulos con carga diferida en segundo plano tan pronto como la aplicación principal se haya iniciado.
