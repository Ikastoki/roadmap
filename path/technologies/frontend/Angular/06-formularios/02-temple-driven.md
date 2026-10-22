<h1 align="center">Template-driven Forms</h1>

<h2>📑 Contenido</h2>

- [Template-driven Forms](#template-driven-forms)
- [Configuración del Módulo](#configuración-del-módulo)
- [Creación del Formulario](#creación-del-formulario)
- [Ejemplo Completo](#ejemplo-completo)

## Template-driven Forms

Los Template-driven forms son una forma de construir formularios donde la lógica y el control del formulario se gestionan principalmente en la plantilla HTML, utilizando directivas de Angular. Este enfoque es ideal para formularios pequeños y simples, ya que la mayor parte de la magia ocurre con un `two-way binding` sencillo.

## Configuración del Módulo

Para que las directivas de formularios, como `ngModel` y `ngForm`, funcionen correctamente, necesitas importar el FormsModule en tu módulo principal (AppModule).

src/app/app.module.ts

```typescript
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { FormsModule } from "@angular/forms"; // Importa FormsModule
import { AppComponent } from "./app.component";

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    FormsModule, // Añádelo al array de imports
  ],
  providers: [],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

## Creación del Formulario

Un formulario controlado por plantilla se crea utilizando una combinación de directivas. La más importante es `ngModel`, que crea un enlace de datos bidireccional entre la propiedad del componente y el valor del campo de entrada.

Elementos clave:

- `form` y `#`: Usamos el `template reference variable` `#miFormulario="ngForm"` para que la directiva ngForm pueda rastrear el estado del formulario (válido, inválido, etc.).

- `ngModel`: Se usa en cada control del formulario (`<input>`, `<select>`, etc.) para vincular el valor a una propiedad del componente. Requiere un atributo name para funcionar correctamente.

- `ngModel` con `two-way binding`: Usando `[(ngModel)]`, logramos que cualquier cambio en el input actualice la propiedad del componente, y viceversa.

- `ngSubmit`: Se usa en el elemento `<form>` para manejar el evento de envío del formulario.

## Ejemplo Completo

Vamos a crear un formulario de registro simple.

src/app/app.component.ts (Componente)

```typescript
import { Component } from "@angular/core";

@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
})
export class AppComponent {
  // Propiedad que el formulario usará para el two-way binding
  usuario = {
    nombre: "",
    email: "",
  };

  onSubmit(formulario: any): void {
    console.log("Formulario enviado:", formulario);
    console.log("Valores del formulario:", this.usuario);
    // Aquí podrías enviar los datos a un servicio
  }
}
```

src/app/app.component.html (Plantilla HTML)

```html
<div
  style="padding: 20px; max-width: 400px; margin: auto; border: 1px solid #ccc; border-radius: 8px;"
>
  <h2>Registro de Usuario</h2>
  <form #registroForm="ngForm" (ngSubmit)="onSubmit(registroForm)">
    <!-- Campo de Nombre -->
    <div class="form-group">
      <label for="nombre">Nombre</label>
      <input
        type="text"
        class="form-control"
        id="nombre"
        name="nombre"
        [(ngModel)]="usuario.nombre"
        required
      />
    </div>

    <!-- Campo de Email -->
    <div class="form-group">
      <label for="email">Email</label>
      <input
        type="email"
        class="form-control"
        id="email"
        name="email"
        [(ngModel)]="usuario.email"
        required
        email
      />
    </div>

    <br />

    <!-- Botón de Envío -->
    <button type="submit" class="btn" [disabled]="!registroForm.valid">
      Registrar
    </button>

    <!-- Muestra el estado del formulario (opcional) -->
    <div *ngIf="registroForm.submitted" style="margin-top: 10px; color: green;">
      Formulario enviado.
    </div>
  </form>
</div>
```
