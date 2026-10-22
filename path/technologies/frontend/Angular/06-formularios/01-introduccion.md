<h1 align="center">Formularios</h1>

<h2>📑 Contenido</h2>

- [Formularios](#formularios)

## Formularios

Los formularios son la forma principal en que los usuarios interactúan con las aplicaciones web, ya sea para iniciar sesión, enviar datos o realizar búsquedas. Angular ofrece dos enfoques para gestionar la lógica y el estado de los formularios: Template-driven forms (formularios controlados por plantillas) y Reactive forms (formularios re

## Template-driven Forms (Formularios controlados por plantillas)

Estos formularios son la forma más sencilla de empezar. Se basan en directivas de Angular que se colocan directamente en la plantilla HTML para construir la estructura del formulario y su lógica.

- ¿Cómo funcionan? Se utilizan directivas como ngModel para crear enlaces de datos bidireccionales (two-way binding) entre el HTML y el modelo del componente. Angular se encarga de la mayor parte de la configuración por ti.

- Ideal para: Formularios simples y básicos, como un formulario de contacto o de inicio de sesión. Son fáciles de entender y de configurar rápidamente.

- Puntos clave: El ngModel se encarga de sincronizar el valor del input con la propiedad usuario.nombre en el componente.

## Reactive Forms (Formularios reactivos)

Los formularios reactivos son una estrategia más escalable y robusta. La lógica y la estructura del formulario se definen de forma programática en el archivo TypeScript del componente, lo que te da un control completo sobre su comportamiento.

- ¿Cómo funcionan? Se utilizan clases de Angular como FormControl, FormGroup y FormArray para construir el modelo del formulario. Se utilizan los métodos valueChanges y statusChanges para observar los cambios en los datos.

- Ideal para: Formularios complejos con lógica de validación avanzada, formularios dinámicos y situaciones donde necesitas mayor capacidad de prueba.

- Puntos clave: El FormGroup agrupa los FormControl (el control nombre en este caso). La validación se define directamente en el archivo TypeScript.

## Validaciones

Ambos tipos de formularios soportan validación, pero la implementación es diferente.

- Template-driven forms: Las validaciones se añaden directamente como atributos en el HTML (ej. `required`, `minlength`).

- Reactive forms: Se utiliza la clase `Validators` en el archivo de TypeScript para añadir validadores (`Validators.required`, `Validators.email`, etc.) al crear los FormControl. Esto ofrece una flexibilidad mucho mayor.
