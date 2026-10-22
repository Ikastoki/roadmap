<h1 align="center">Módulos</h1>

<h2>📑 Contenido</h2>

- [Módulos](#módulos)
- [¿Qué es un NgModule?](#qué-es-un-ngmodule)
- [La importancia de los módulos](#la-importancia-de-los-módulos)

## Módulos

Los módulos (`NgModule`) son la forma en que Angular organiza la aplicación. Funcionan como contenedores lógicos que agrupan componentes, servicios, directivas y pipes relacionados en bloques coherentes y funcionales.

## ¿Qué es un NgModule?

Cada aplicación de Angular tiene al menos un módulo raíz, llamado `AppModule`, que se genera por defecto. Este módulo es el punto de entrada de la aplicación.

Un módulo se define con el decorador `@NgModule`, que es un objeto de configuración con las siguientes propiedades clave:

- `declarations`: Un array donde se declaran los componentes, directivas y pipes que pertenecen a este módulo.

- `imports`: Un array con los módulos que este módulo necesita para funcionar. Por ejemplo, BrowserModule para aplicaciones web, o `FormsModule` para trabajar con formularios.

- `providers`: Un array de los servicios que estarán disponibles para los componentes de este módulo. Aquí se configura la Inyección de Dependencias.

- `bootstrap`: Solo se usa en el módulo raíz (`AppModule`). Aquí se declara el componente que debe ser el componente principal de la aplicación, el que se carga primero al iniciar (generalmente, `AppComponent`).

## La importancia de los módulos

Los módulos son fundamentales para:

- Organización: Mantienen el código estructurado y modular, haciendo que las aplicaciones grandes sean más fáciles de gestionar.

- Encapsulamiento: Permiten agrupar funcionalidades relacionadas, como un módulo de "autenticación" o un módulo de "usuarios", facilitando la reutilización del código.

- Lazy Loading (Carga perezosa): Permiten cargar módulos solo cuando son necesarios, lo que optimiza el rendimiento de la aplicación al reducir el tiempo de carga inicial.
