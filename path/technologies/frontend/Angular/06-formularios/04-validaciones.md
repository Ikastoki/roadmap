<h1 align="center">Validaciones</h1>

<h2>📑 Contenido</h2>

- [Validaciones](#validaciones)
- [Validadores Integrados (Validators)](#validadores-integrados-validators)
- [Mensajes de Error en la Plantilla](#mensajes-de-error-en-la-plantilla)
- [Validadores Personalizados](#validadores-personalizados)
  - [Ejemplo: Validador para verificar que un nombre de usuario no contenga la palabra "admin"](#ejemplo-validador-para-verificar-que-un-nombre-de-usuario-no-contenga-la-palabra-admin)

## Validaciones

La validación es un aspecto crítico de los formularios para garantizar que los datos que el usuario introduce son correctos. En los formularios reactivos, la validación se gestiona de forma programática en el archivo TypeScript del componente, lo que ofrece un control total y una gran capacidad de prueba.

## Validadores Integrados (Validators)

Angular ofrece una clase Validators con una serie de validadores predefinidos que puedes usar. Simplemente los añades al FormControl al inicializarlo.

Aquí tienes algunos de los más comunes:

| Validador                   | Descripción                                                            | Ejemplo                                                |
| --------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------ |
| `Validators.required`       | El campo no puede estar vacío.                                         | `new FormControl('', Validators.required)`             |
| `Validators.email`          | El valor debe tener el formato de una dirección de correo electrónico. | `new FormControl('', Validators.email)`                |
| `Validators.minLength(n)`   | La longitud mínima del valor es `n`.                                   | `new FormControl('', Validators.minLength(8))`         |
| `Validators.maxLength(n)`   | La longitud máxima del valor es `n`.                                   | `new FormControl('', Validators.maxLength(50))`        |
| `Validators.pattern(regex)` | El valor debe coincidir con una expresión regular.                     | `new FormControl('', Validators.pattern('[a-zA-Z]*'))` |
| `Validators.min(n)`         | El valor numérico debe ser mayor o igual a `n`.                        | `new FormControl(18, Validators.min(18))`              |
| `Validators.max(n)`         | El valor numérico debe ser menor o igual a `n`.                        | `new FormControl(100, Validators.max(150))`            |

Puedes pasar varios validadores en un array para un solo control.

```typescript
// Ejemplo de un campo de contraseña con múltiples validadores
'password': new FormControl(null, [
  Validators.required,
  Validators.minLength(8),
  Validators.pattern('^(?=.*[0-9])(?=.*[a-zA-Z])([a-zA-Z0-9]+)$')
])
```

## Mensajes de Error en la Plantilla

Para mostrar mensajes de error útiles al usuario, puedes acceder a las propiedades de error de un FormControl en la plantilla HTML.

```html
<!-- Campo de Email -->
<div class="form-group">
  <label for="email">Email</label>
  <input type="email" id="email" formControlName="email" />

  <!-- Contenedor para los mensajes de error -->
  <div
    *ngIf="formularioRegistro.get('email').invalid && formularioRegistro.get('email').touched"
  >
    <small
      style="color: red;"
      *ngIf="formularioRegistro.get('email').errors.required"
    >
      El email es obligatorio.
    </small>
    <small
      style="color: red;"
      *ngIf="formularioRegistro.get('email').errors.email"
    >
      Por favor, introduce un email válido.
    </small>
  </div>
</div>
```

En este ejemplo, usamos `*ngIf` para comprobar si hay un error específico (`errors.required`, `errors.email`) y mostramos el mensaje correspondiente.

## Validadores Personalizados

A veces, los validadores integrados no son suficientes. Puedes crear tus propios validadores personalizados, que no son más que funciones.

Un validador personalizado es una función que recibe un `FormControl` y retorna un objeto de error (si la validación falla) o `null` (si la validación es exitosa).

### Ejemplo: Validador para verificar que un nombre de usuario no contenga la palabra "admin"

src/app/custom-validators.ts (Archivo de validadores)

```typescript
import { AbstractControl, ValidationErrors } from "@angular/forms";

export class CustomValidators {
  static noAdmin(control: AbstractControl): ValidationErrors | null {
    if (
      control.value &&
      (control.value as string).toLowerCase().includes("admin")
    ) {
      // Retorna un objeto de error si la validación falla
      return { noAdmin: true };
    }
    // Retorna null si la validación es exitosa
    return null;
  }
}
```

Uso del validador personalizado en un componente:

```typescript
// En src/app/app.component.ts
import { Component, OnInit } from '@angular/core';
import { FormGroup, FormControl, Validators } from '@angular/forms';
import { CustomValidators } from './custom-validators'; // Importa el validador

@Component({ ... })
export class AppComponent implements OnInit {
  formularioRegistro: FormGroup;

  ngOnInit(): void {
    this.formularioRegistro = new FormGroup({
      'username': new FormControl(null, [
        Validators.required,
        CustomValidators.noAdmin // Usa el validador personalizado
      ]),
      // ... otros campos ...
    });
  }
}
```
