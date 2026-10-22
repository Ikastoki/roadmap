<h1 align="center">Inyección de Dependencias (ID)</h1>

<h2>📑 Contenido</h2>

- [Inyección de Dependencias (ID)](#inyección-de-dependencias-id)
- [Conceptos Clave de la Inyección de Dependencias](#conceptos-clave-de-la-inyección-de-dependencias)
- [¿Cómo funciona en la práctica?](#cómo-funciona-en-la-práctica)

## Inyección de Dependencias (ID)

La Inyección de Dependencias (ID) es un patrón de diseño fundamental en Angular que se utiliza para desacoplar el código. En lugar de que un componente cree sus propias dependencias (como un servicio), Angular se las proporciona automáticamente.

Piénsalo así: si un componente necesita un servicio para obtener datos, en lugar de que el componente sea responsable de crear ese servicio, simplemente le dice a Angular "necesito este servicio", y Angular se lo entrega. Esto hace que el código sea más limpio, más fácil de probar y más reutilizable.

## Conceptos Clave de la Inyección de Dependencias

Para entender cómo funciona la ID, hay tres elementos principales que debes conocer:

- **Inyector** (`Injector`): El Injector es un mecanismo interno de Angular que se encarga de crear las instancias de los servicios y entregarlas cuando se solicitan. Cada componente y módulo tiene su propio inyector, formando una jerarquía.

- **Proveedor** (`Provider`): El Provider es la "receta" que le dice al inyector cómo crear una instancia de un servicio. Se define en los metadatos del decorador `@Injectable` de un servicio, o en el providers array de un `@NgModule` o `@Component`.

- **Dependencia**: La dependencia es la clase o servicio que una clase necesita para funcionar. En Angular, un componente solicita una dependencia a través de su constructor.

## ¿Cómo funciona en la práctica?

El proceso de inyección de dependencias es bastante sencillo. Por ejemplo, si tienes un componente que necesita un servicio llamado `UserService`, solo tienes que declararlo en el constructor de tu componente.

```typescript
import { Component } from "@angular/core";
import { UserService } from "./user.service"; // Importamos el servicio

@Component({
  selector: "app-mi-componente",
  templateUrl: "./mi-componente.component.html",
})
export class MiComponente {
  // Angular detecta que este componente necesita UserService
  // y lo inyecta automáticamente.
  constructor(private userService: UserService) {}

  // Ahora puedes usar 'this.userService' en tu componente
  obtenerUsuarios() {
    this.userService.getUsuarios();
  }
}
```

En este ejemplo, Angular ve que MiComponente necesita UserService y, gracias a los providers, el inyector se encarga de crear una única instancia de UserService y la asigna a la propiedad userService del componente.
