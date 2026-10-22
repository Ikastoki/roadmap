<h1 align="center">Compartir datos entre componentes</h1>

<h2>📑 Contenido</h2>

- [Compartir datos entre componentes](#compartir-datos-entre-componentes)
- [Comunicación de Padre a Hijo: `@Input()`](#comunicación-de-padre-a-hijo-input)
- [Comunicación de Hijo a Padre: @Output()](#comunicación-de-hijo-a-padre-output)
- [Comunicación entre Componentes No Relacionados: Servicios](#comunicación-entre-componentes-no-relacionados-servicios)

## Compartir datos entre componentes

Existen varias formas de que los componentes de Angular se comuniquen entre sí. La elección del método depende de la relación que exista entre ellos (parental, no relacionada, etc.).

## Comunicación de Padre a Hijo: `@Input()`

Cuando un componente padre necesita pasar datos a un componente hijo, se utiliza el decorador `@Input()`. Es el método más común y sencillo para esta dirección de flujo.

- ¿Cómo funciona? El componente hijo declara una propiedad con `@Input()`, lo que la convierte en una propiedad de entrada que puede ser "set" por el componente padre.

- Ejemplo: Imagina un componente `tarjeta-producto` que necesita recibir la información del producto a mostrar.

tarjeta-producto.component.ts (Hijo)

```typescript
import { Component, Input } from "@angular/core";

@Component({
  selector: "app-tarjeta-producto",
  template: `
    <div class="card">
      <h3>{{ producto.nombre }}</h3>
      <p>Precio: {{ producto.precio | currency }}</p>
    </div>
  `,
  styleUrls: ["./tarjeta-producto.component.css"],
})
export class TarjetaProductoComponent {
  @Input() producto: any;
}
```

app.component.ts (Padre)

```typescript
import { Component } from "@angular/core";

@Component({
  selector: "app-root",
  template: `
    <div style="padding: 20px;">
      <app-tarjeta-producto [producto]="miProducto"></app-tarjeta-producto>
    </div>
  `,
})
export class AppComponent {
  miProducto = { nombre: "Libro de Angular", precio: 29.99 };
}
```

En este caso, el padre (`AppComponent`) pasa el objeto `miProducto` al hijo (`TarjetaProductoComponent`) a través del binding de propiedades `[producto]`.

## Comunicación de Hijo a Padre: @Output()

Cuando un componente hijo necesita enviar datos o notificar eventos a su componente padre, se utiliza el decorador `@Output()` junto con `EventEmitter`.

- ¿Cómo funciona? El componente hijo crea un `EventEmitter` con `@Output()` para emitir un evento. El componente padre "escucha" este evento y ejecuta una función cuando se dispara.

- Ejemplo: Imagina que el componente hijo `tarjeta-producto` tiene un botón que, al ser pulsado, notifica al padre que se ha agregado un producto.

tarjeta-producto.component.ts (Hijo)

```typescript
import { Component, Output, EventEmitter } from "@angular/core";

@Component({
  selector: "app-tarjeta-producto",
  template: `
    <div class="card">
      <p>... Detalles del producto ...</p>
      <button (click)="agregarAlCarrito()">Agregar al carrito</button>
    </div>
  `,
})
export class TarjetaProductoComponent {
  @Output() productoAgregado = new EventEmitter<string>();

  agregarAlCarrito() {
    this.productoAgregado.emit("Se ha agregado un producto"); // Emite un evento
  }
}
```

app.component.ts (Padre)

```typescript
import { Component } from "@angular/core";

@Component({
  selector: "app-root",
  template: `
    <div style="padding: 20px;">
      <app-tarjeta-producto
        (productoAgregado)="mostrarMensaje($event)"
      ></app-tarjeta-producto>
    </div>
  `,
})
export class AppComponent {
  mostrarMensaje(mensaje: string) {
    alert(mensaje);
  }
}
```

El padre (`AppComponent`) se suscribe al evento (`productoAgregado`) del hijo. Cuando el hijo lo emite, el padre ejecuta el método mostrarMensaje con los datos recibidos.

## Comunicación entre Componentes No Relacionados: Servicios

Cuando dos componentes no tienen una relación de padre e hijo, la mejor manera de compartir datos es a través de un servicio compartido.

- ¿Cómo funciona? Se crea un servicio inyectable que actúa como un almacén de datos. Los componentes que necesitan datos pueden inyectar el servicio y suscribirse a un `Observable` (generalmente un `Subject` o `BehaviorSubject`) dentro del servicio para recibir actualizaciones en tiempo real.

- Ejemplo: Un componente `navbar` y un componente `shopping-cart` necesitan saber cuántos artículos hay en el carrito, pero no están directamente relacionados.

carrito.service.ts

```typescript
import { Injectable } from "@angular/core";
import { Subject } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class CarritoService {
  private _contador = 0;
  private carritoSubject = new Subject<number>();

  // Observable para que otros componentes se suscriban
  public get contador$() {
    return this.carritoSubject.asObservable();
  }

  agregarProducto() {
    this._contador++;
    this.carritoSubject.next(this._contador); // Emite el nuevo valor
  }
}
```

Componente que agrega productos

```typescript
// Componente A
import { Component } from '@angular/core';
import { CarritoService } from '../carrito.service';

@Component({...})
export class ProductListComponent {
  constructor(private carritoService: CarritoService) { }

  onAgregarClick() {
    this.carritoService.agregarProducto();
  }
}
```

Componente que muestra el conteo

```typescript
// Componente B
import { Component, OnInit } from '@angular/core';
import { CarritoService } from '../carrito.service';

@Component({...})
export class NavbarComponent implements OnInit {
  conteo: number = 0;

  constructor(private carritoService: CarritoService) { }

  ngOnInit(): void {
    // Se suscribe al observable para recibir actualizaciones
    this.carritoService.contador$.subscribe(nuevoConteo => {
      this.conteo = nuevoConteo;
    });
  }
}
```
