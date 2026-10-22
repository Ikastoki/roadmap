<h1 align="center">Reactive Forms </h1>

<h2>📑 Contenido</h2>

- [Reactive Forms](#reactive-forms)
- [Configuración del Módulo](#configuración-del-módulo)
- [Componentes Clave de los Formularios Reactivos](#componentes-clave-de-los-formularios-reactivos)
- [Ejemplo Completo: Un Formulario de Registro](#ejemplo-completo-un-formulario-de-registro)

## Reactive Forms

Los Reactive forms son una forma de construir formularios de manera programática en Angular. A diferencia de los formularios template-driven, toda la lógica y la estructura del formulario se definen en el archivo TypeScript del componente, lo que ofrece mayor flexibilidad, escalabilidad y capacidad de prueba.

## Configuración del Módulo

Para usar formularios reactivos, debes importar el `ReactiveFormsModule` en tu módulo principal (`AppModule`).

src/app/app.module.ts

```typescript
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { FormsModule, ReactiveFormsModule } from "@angular/forms"; // Importa ReactiveFormsModule
import { AppComponent } from "./app.component";

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    FormsModule,
    ReactiveFormsModule, // Añádelo al array de imports
  ],
  providers: [],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

## Componentes Clave de los Formularios Reactivos

Los formularios reactivos se construyen alrededor de tres clases principales:

- FormControl: Representa un solo campo de entrada de formulario, como un `<input>` o un `<textarea>`. Es la unidad más básica que rastrea el valor y el estado de validación de un elemento de la interfaz.

- FormGroup: Es una colección de `FormControl`. Agrupa múltiples controles para que puedas gestionarlos como una sola unidad. Un `FormGroup` también rastrea el estado y el valor de sus controles hijos.

- Validators: Una clase que proporciona funciones de validación integradas. Por ejemplo, `Validators.required` asegura que un campo no esté vacío.

## Ejemplo Completo: Un Formulario de Registro

Vamos a crear un formulario de registro simple para ver cómo se conectan estas piezas.

src/app/app.component.ts (Componente)

```typescript
import { Component, OnInit } from "@angular/core";
import { FormGroup, FormControl, Validators } from "@angular/forms";

@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
})
export class AppComponent implements OnInit {
  // Define un FormGroup para agrupar los controles
  formularioRegistro: FormGroup;

  ngOnInit(): void {
    // Inicializa el formulario y sus controles
    this.formularioRegistro = new FormGroup({
      // Define un FormControl para el campo 'nombre'
      // El primer argumento es el valor inicial (null)
      // El segundo es un array de validadores
      nombre: new FormControl(null, [Validators.required]),

      // Define un FormControl para el campo 'email' con dos validadores
      email: new FormControl(null, [Validators.required, Validators.email]),
    });
  }

  onSubmit(): void {
    console.log("Formulario enviado:", this.formularioRegistro.value);
    console.log("¿Es válido?:", this.formularioRegistro.valid);
  }
}
```

src/app/app.component.html (Plantilla HTML)

```html
<div
  style="padding: 20px; max-width: 400px; margin: auto; border: 1px solid #ccc; border-radius: 8px;"
>
  <h2>Registro de Usuario</h2>
  <!--
    Se enlaza el formulario con la propiedad 'formularioRegistro'
    del componente usando la directiva [formGroup]
  -->
  <form [formGroup]="formularioRegistro" (ngSubmit)="onSubmit()">
    <!-- Campo de Nombre -->
    <div class="form-group">
      <label for="nombre">Nombre</label>
      <!--
        Se enlaza el input con el FormControl 'nombre'
        usando la directiva formControlName
      -->
      <input
        type="text"
        class="form-control"
        id="nombre"
        formControlName="nombre"
      />
      <!-- Mensaje de error si el campo es inválido y ha sido tocado -->
      <div
        *ngIf="formularioRegistro.get('nombre').invalid && formularioRegistro.get('nombre').touched"
      >
        <small style="color: red;">El nombre es obligatorio.</small>
      </div>
    </div>

    <!-- Campo de Email -->
    <div class="form-group">
      <label for="email">Email</label>
      <input
        type="email"
        class="form-control"
        id="email"
        formControlName="email"
      />
      <!-- Mensaje de error si el campo es inválido y ha sido tocado -->
      <div
        *ngIf="formularioRegistro.get('email').invalid && formularioRegistro.get('email').touched"
      >
        <small style="color: red;"
          >El email es obligatorio y debe ser válido.</small
        >
      </div>
    </div>

    <br />

    <!-- Botón de Envío, deshabilitado si el formulario no es válido -->
    <button type="submit" class="btn" [disabled]="!formularioRegistro.valid">
      Registrar
    </button>
  </form>
</div>
```
