<h1 align="center">Servicios</h1>

<h2>📑 Contenido</h2>

- [Servicios](#servicios)
- [¿Por qué usar servicios?](#por-qué-usar-servicios)
- [Creación y estructura](#creación-y-estructura)
- [Uso de servicios](#uso-de-servicios)

## Servicios

Los servicios son clases de Angular que se utilizan para encapsular y compartir lógica y datos a lo largo de toda la aplicación. A diferencia de los componentes, los servicios no están atados a una vista (template HTML). Su propósito es ser inyectados y reutilizados por múltiples componentes.

## ¿Por qué usar servicios?

- Separación de responsabilidades: Mantienen la lógica de tu aplicación separada de la vista. Los componentes se centran en interactuar con el usuario y mostrar datos, mientras que los servicios se encargan de obtener esos datos, manipularlos y realizar otras tareas.

- Reusabilidad: Puedes crear un solo servicio y reutilizarlo en múltiples componentes sin tener que duplicar el código.

- Gestión de datos: Son ideales para almacenar y compartir datos entre componentes que no tienen una relación directa (padre-hijo).

## Creación y estructura

Al igual que con otros elementos de Angular, la mejor manera de crear un servicio es usando el Angular CLI.

```bash
ng generate service nombre-del-servicio
# O la versión corta
ng g s nombre-del-servicio
```

Esto generará un archivo llamado `nombre-del-servicio.service.ts` con la siguiente estructura:

```typescript
import { Injectable } from "@angular/core";

@Injectable({
  providedIn: "root",
})
export class NombreDelServicioService {
  constructor() {}
}
```

- `@Injectable()`: Este decorador es clave. Le indica a Angular que esta clase es un servicio y que puede ser inyectada en otras clases.

- `providedIn: 'root'`: Esta es la forma más común de registrar un servicio. Le dice a Angular que el servicio debe ser un singleton (una única instancia) y que debe estar disponible en toda la aplicación.

## Uso de servicios

Una vez que has creado un servicio, puedes inyectarlo en cualquier componente, directiva o incluso en otro servicio. Como vimos en la introducción a la Inyección de Dependencias, esto se hace a través del constructor.

```typescript
import { Component } from "@angular/core";
import { NombreDelServicioService } from "./nombre-del-servicio.service";

@Component({
  selector: "app-mi-componente",
  templateUrl: "./mi-componente.component.html",
})
export class MiComponente {
  // Angular inyecta automáticamente el servicio en el constructor
  constructor(private servicio: NombreDelServicioService) {}

  // Ahora puedes usar el servicio para llamar a sus métodos
  algunMetodo() {
    this.servicio.algunMetodoDelServicio();
  }
}
```
