<h1 align="center">Parámetros de ruta </h1>

<h2>📑 Contenido</h2>

- [Parámetros de ruta](#parámetros-de-ruta)
- [¿Qué son y cómo se definen?](#qué-son-y-cómo-se-definen)
- [Acceder al valor del parámetro en un componente](#acceder-al-valor-del-parámetro-en-un-componente)
  - [Usando snapshot (Captura única)](#usando-snapshot-captura-única)
  - [Usando Observable (La forma recomendada)](#usando-observable-la-forma-recomendada)
- [Ejemplo completo](#ejemplo-completo)

## Parámetros de ruta

Los parámetros de ruta (o route parameters) son una característica poderosa del enrutador de Angular que te permite capturar valores dinámicos de la URL. Esto es esencial para crear vistas que dependen de datos específicos, como una página de detalles de un producto (`/productos/123`) o un perfil de usuario (`/usuarios/juan`).

## ¿Qué son y cómo se definen?

Un parámetro de ruta es un segmento de la URL que actúa como un marcador de posición para un valor. Se define en la configuración de la ruta usando dos puntos : seguidos del nombre del parámetro.

```typescript
const routes: Routes = [
  { path: "productos", component: ProductListComponent },
  { path: "productos/:id", component: ProductDetailComponent }, // El ':id' es el parámetro de ruta
];
```

En este ejemplo, cuando el usuario navegue a una URL como `/productos/123`, el valor 123 se capturará como el parámetro `id`.

## Acceder al valor del parámetro en un componente

Para obtener el valor del parámetro en tu componente, debes inyectar el servicio `ActivatedRoute`. Este servicio contiene información sobre la ruta actual y te da acceso a los parámetros. Hay dos formas principales de hacerlo:

### Usando snapshot (Captura única)

El `snapshot` te da acceso a los parámetros de la ruta en un momento específico (cuando el componente se inicializa por primera vez). Es útil si sabes que el componente no volverá a usar la misma ruta con un parámetro diferente (ej. nunca navegarás de `/productos/1` a `/productos/2` sin recargar el componente).

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({ ... })
export class ProductDetailComponent implements OnInit {
  productId: string;

  constructor(private route: ActivatedRoute) { }

  ngOnInit(): void {
    // Acceder al parámetro 'id' usando el snapshot
    this.productId = this.route.snapshot.paramMap.get('id');
    console.log('ID del producto (snapshot):', this.productId);
  }
}
```

### Usando Observable (La forma recomendada)

La forma más robusta es suscribirse al observable `paramMap` de `ActivatedRoute`. Esto te permite reaccionar a los cambios de parámetros sin que el componente sea destruido y recreado. Por ejemplo, si un usuario navega de `/productos/1` a `/productos/2` a través de un enlace dentro del mismo componente.

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({ ... })
export class ProductDetailComponent implements OnInit {
  productId: string;

  constructor(private route: ActivatedRoute) { }

  ngOnInit(): void {
    // Suscribirse al observable para reaccionar a los cambios de parámetros
    this.route.paramMap.subscribe(params => {
      this.productId = params.get('id');
      console.log('ID del producto (observable):', this.productId);
      // Aquí puedes cargar los datos del producto con este ID
    });
  }
}
```

## Ejemplo completo

Vamos a simular un caso práctico con una ruta y un componente.

app-routing.module.ts

```typescript
import { NgModule } from "@angular/core";
import { Routes, RouterModule } from "@angular/router";
import { ProductDetailComponent } from "./product-detail/product-detail.component";

const routes: Routes = [
  // Ruta con un parámetro llamado 'id'
  { path: "productos/:id", component: ProductDetailComponent },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

product-detail.component.ts

```typescript
import { Component, OnInit } from "@angular/core";
import { ActivatedRoute } from "@angular/router";

@Component({
  selector: "app-product-detail",
  template: `
    <div
      style="text-align: center; padding: 20px; border: 1px solid #ccc; border-radius: 8px;"
    >
      <h2>Detalles del Producto</h2>
      <p>
        El ID del producto es: <strong>{{ productId }}</strong>
      </p>
    </div>
  `,
  styles: [
    `
      h2 {
        color: #333;
      }
      p {
        font-size: 1.2em;
      }
    `,
  ],
})
export class ProductDetailComponent implements OnInit {
  productId: string | null = null;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Usamos el snapshot para un ejemplo sencillo
    this.productId = this.route.snapshot.paramMap.get("id");
  }
}
```
