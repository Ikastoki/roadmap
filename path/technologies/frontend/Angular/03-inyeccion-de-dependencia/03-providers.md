<h1 align="center">Providers</h1>

<h2>📑 Contenido</h2>

- [Providers](#providers)
- [¿Dónde se definen los providers?](#dónde-se-definen-los-providers)
  - [En el decorador @Injectable() del servicio (Método recomendado)](#en-el-decorador-injectable-del-servicio-método-recomendado)
  - [En el array providers de un @NgModule o @Component (Método tradicional)](#en-el-array-providers-de-un-ngmodule-o-component-método-tradicional)

## Providers

Los providers son la configuración que le dice al inyector de Angular cómo crear o encontrar la instancia de una dependencia (como un servicio). En esencia, un provider es la "receta" que Angular utiliza para entregar una dependencia cuando un componente la solicita a través de su constructor.

## ¿Dónde se definen los providers?

Hay dos lugares principales para configurar un provider y, dependiendo de dónde lo hagas, la dependencia tendrá un alcance diferente.

### En el decorador @Injectable() del servicio (Método recomendado)

Esta es la forma más común y eficiente de registrar un servicio. Al usar la propiedad `providedIn: 'root'`, le dices a Angular que este servicio debe ser un singleton para toda la aplicación. Angular se encarga de crear una única instancia que estará disponible para cualquier componente que la necesite.

- Ventaja clave: Esta configuración es tree-shakable, lo que significa que si el servicio no es utilizado en tu aplicación, el compilador lo eliminará del bundle final, optimizando el rendimiento.

  ```typescript
  // En user.service.ts
  import { Injectable } from "@angular/core";

  @Injectable({
    providedIn: "root", // Este es el provider
  })
  export class UserService {
    // ...
  }
  ```

### En el array providers de un @NgModule o @Component (Método tradicional)

Aunque el método anterior es el preferido, también puedes registrar un provider manualmente en un módulo o componente.

- Registro en un Módulo (AppModule): La instancia del servicio estará disponible para todos los componentes de ese módulo. La mayoría de las veces, si no usas `providedIn: 'root'`, registrarás los servicios aquí.

  ```typescript
  // En app.module.ts
  import { UserService } from "./user.service";

  @NgModule({
    declarations: [AppComponent],
    imports: [BrowserModule],
    providers: [UserService], // El servicio está disponible en este módulo
    bootstrap: [AppComponent],
  })
  export class AppModule {}
  ```

- Registro en un Componente: Si registras el servicio en un componente, se crea una nueva instancia del servicio cada vez que se crea una instancia de ese componente. Esto es útil para servicios que deben mantener un estado local y no deben ser compartidos.

  ```typescript
  // En mi-componente.component.ts
  import { UserService } from "./user.service";

  @Component({
    selector: "app-mi-componente",
    templateUrl: "./mi-componente.component.html",
    providers: [UserService], // Una nueva instancia para cada MiComponente
  })
  export class MiComponente {
    // ...
  }
  ```
