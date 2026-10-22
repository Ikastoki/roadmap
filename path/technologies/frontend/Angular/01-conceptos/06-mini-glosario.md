<h1 align="center">Mini Glosario</h1>

<h2>📑 Contenido</h2>

- [](#)

## Componentes

Los componentes son los bloques de construcción básicos de cualquier aplicación de Angular. Cada componente es una clase de TypeScript que controla una parte de la pantalla (la vista) y contiene su propia lógica y datos. Un componente es el elemento central que agrupa la lógica (la clase de TypeScript), la vista (el template HTML) y los estilos (CSS).

## Templates

El template es el HTML de un componente. Es la parte que define cómo se renderizará el componente en el navegador. En el template, puedes usar la sintaxis especial de Angular para mostrar datos del componente, manejar eventos o manipular el DOM de forma dinámica.

## Módulos (NgModule)

Los módulos son la forma en que Angular organiza la aplicación. Un módulo agrupa componentes, directivas, pipes y servicios relacionados. Cada aplicación de Angular tiene al menos un módulo raíz, llamado `AppModule`, que es el punto de entrada. Los módulos te permiten mantener la aplicación organizada, facilitando la escalabilidad y la carga perezosa (lazy loading).

## Directivas

Las directivas son clases de TypeScript que añaden o modifican el comportamiento de los elementos del DOM. Angular incluye una serie de directivas predefinidas, y también puedes crear las tuyas propias. Existen tres tipos principales:

- Directivas de componente: Son directivas con su propio template. (Aunque se llaman componentes, técnicamente son un tipo de directiva).

- Directivas estructurales: Modifican la estructura del DOM añadiendo, eliminando o manipulando elementos (ej. `*ngIf`, `*ngFor`).

- Directivas de atributo: Cambian la apariencia o el comportamiento de un elemento, componente o de otra directiva (ej. `[ngClass]`, `[ngStyle]`).

## Decoradores y Metadata

En Angular, utilizamos decoradores para añadir metadata a las clases de TypeScript. Los decoradores son funciones que se aplican a clases, métodos o propiedades para modificar su comportamiento o para proporcionarle información a Angular sobre cómo deben ser tratados.

El decorador más común es `@Component`. Cuando defines un componente, le añades este decorador a la clase. La metadata que pasas dentro de `@Component` le dice a Angular detalles importantes, como:

- `selector`: El nombre de la etiqueta HTML (`<app-mi-componente>`) que usarás para insertar este componente en tu aplicación.

- `templateUrl`: La ruta al archivo HTML del template.

- `styleUrls`: La ruta al archivo de estilos CSS/Sass.

Otros decoradores importantes que verás son `@NgModule` para los módulos y `@Injectable` para los servicios.

## Data Binding (Enlace de datos)

El data binding es la comunicación entre la lógica de tu componente (la clase de TypeScript) y su template (el HTML). Es el mecanismo que permite que los datos fluyan en tu aplicación. Se divide en dos tipos principales:

- **One-way data binding**: El flujo de datos va en una única dirección. Puede ser desde la clase al template (para mostrar datos) o desde el template a la clase (para manejar eventos del usuario).

- **Two-way data binding**: El flujo de datos es bidireccional, es decir, cualquier cambio en el template actualiza la clase y viceversa.

## Inyección de Dependencias (DI)

La Inyección de Dependencias es un patrón de diseño fundamental en Angular. Permite que una clase (como un componente) obtenga las dependencias que necesita (como un servicio) sin tener que crearlas por sí misma. Angular se encarga de "inyectar" estas dependencias de manera automática.

Gracias a la DI, puedes reutilizar servicios, probar componentes de forma más sencilla y mantener el código más limpio y modular.
