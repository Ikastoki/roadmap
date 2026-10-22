<h1 align="center">Angular CLI</h1>

<h2>📑 Contenido</h2>

- [Angular CLI](#angular-cli)
- [Comandos más utilizados](#comandos-más-utilizados)
- [Generar archivos](#generar-archivos)
- [Opciones para personalizar la generación](#opciones-para-personalizar-la-generación)

## Angular CLI

El Angular CLI (Command Line Interface) es una herramienta de línea de comandos que se utiliza para inicializar, desarrollar, mantener y generar aplicaciones de Angular. Facilita la creación de proyectos, componentes, servicios y otros elementos, automatizando gran parte del proceso de configuración.

## Comandos más utilizados

| Comando     | Descripción                                                                | Ejemplo de uso                      |
| ----------- | -------------------------------------------------------------------------- | ----------------------------------- |
| ng new      | Crea un nuevo espacio de trabajo y una aplicación de Angular.              | ng new mi-app                       |
| ng serve    | Construye y sirve la aplicación, recargando automáticamente al guardar.    | ng serve                            |
| ng generate | Genera y/o modifica archivos para componentes, servicios, módulos, etc.    | ng generate component mi-componente |
| ng test     | Ejecuta las pruebas unitarias de la aplicación.                            | ng test                             |
| ng e2e      | Ejecuta las pruebas end-to-end de la aplicación.                           | ng e2e                              |
| ng build    | Compila la aplicación en un directorio de salida para su despliegue.       | ng build                            |
| ng lint     | Analiza el código para encontrar errores y problemas de estilo.            | ng lint                             |
| ng update   | Actualiza la aplicación y sus dependencias a la última versión de Angular. | ng update                           |
| ng add      | Añade soporte para una librería externa en tu proyecto.                    | ng add @angular/pwa                 |
| ng help     | Muestra una lista de comandos y sus opciones.                              | ng help                             |

## Generar archivos

El comando `ng generate` (o su alias `ng g`) es una herramienta fundamental en Angular CLI. Permite generar diferentes tipos de archivos y estructuras de código, lo que acelera el desarrollo y asegura una convención de nombres y una estructura consistente en todo el proyecto.

- `component` (o `c`): Crea un nuevo componente. Esto incluye el archivo de TypeScript (.ts), el template HTML (.html), el archivo de estilos (.css/scss/etc.) y un archivo de pruebas (.spec.ts). Por ejemplo: `ng g component users/user-list`.

- `directive` (o `d`): Genera una directiva, la cual te permite modificar la apariencia o el comportamiento de elementos del DOM.

- `service` (o `s`): Crea un servicio para compartir lógica y datos a través de la aplicación.

- `pipe` (o `p`): Genera un pipe para transformar datos en las plantillas HTML.

- `module` (o `m`): Crea un módulo (NgModule) para agrupar componentes, servicios y otros elementos relacionados.

- `class`: Genera una clase genérica.

- `guard`: Crea un guard de ruta, que se usa para controlar el acceso a ciertas rutas.

- `interface`: Genera una interfaz de TypeScript.

- `enum`: Crea un enum de TypeScript.

## Opciones para personalizar la generación

Puedes usar flags (banderas) para modificar el comportamiento de `ng generate`. Aquí tienes algunas de las opciones más útiles:

- `--skip-tests`: Esta bandera es muy útil para evitar la creación del archivo de pruebas (.spec.ts) para el elemento que estás generando. Es ideal si no planeas escribir pruebas unitarias de inmediato o si las gestionarás de forma diferente. Por ejemplo: `ng g component profile --skip-tests`.

- `--inline-style` (o `-s`): Evita que se cree un archivo de estilos separado (.css, .scss, etc.) para el componente. En su lugar, el CSS se inyecta directamente en el archivo .ts del componente.

- `--inline-template` (o `-t`): Evita que se cree un archivo HTML separado. El HTML se inyecta directamente en el archivo .ts del componente.

- `--style` (o `-style`): Permite especificar el preprocesador CSS que quieres usar para el componente, como scss o sass. Por ejemplo: `ng g component new-page --style scss`.

- `--dry-run` (o `-d`): Muestra en la consola los archivos que se generarían sin crearlos físicamente. Esto es perfecto para verificar que el comando hará lo que esperas antes de ejecutarlo realmente.
