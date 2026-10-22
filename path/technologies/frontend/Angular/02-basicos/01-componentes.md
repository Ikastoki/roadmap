<h1 align="center">Componentes</h1>

<h2>📑 Contenido</h2>

- [Componentes](#componentes)
- [¿Cómo crear un componente?](#cómo-crear-un-componente)
- [Estructura de un componente](#estructura-de-un-componente)

## Componentes

Los componentes son los bloques de construcción fundamentales de una aplicación de Angular. Son la combinación de una plantilla (HTML), una hoja de estilos (CSS) y una clase (TypeScript). Cada componente controla una parte específica de la interfaz de usuario, y juntos, forman la jerarquía completa de tu aplicación.

## ¿Cómo crear un componente?

La mejor y más recomendada forma de crear un componente es usando el Angular CLI. Esto asegura que los archivos se generen con la estructura correcta y que se incluyan automáticamente en el módulo de tu aplicación.

Para crear un nuevo componente, usa el siguiente comando en tu terminal:

```bash
ng generate component nombre-del-componente
# O la versión corta
ng g c nombre-del-componente
```

## Estructura de un componente

Cuando ejecutas el comando `ng generate`, se crea una carpeta con el nombre del componente y dentro, se generan cuatro archivos principales. Usando el ejemplo de perfil-usuario, la estructura sería:

```
perfil-usuario/
├── perfil-usuario.component.ts
├── perfil-usuario.component.html
├── perfil-usuario.component.css
└── perfil-usuario.component.spec.ts
```

- `perfil-usuario.component.ts` (La Lógica): Es la clase de TypeScript del componente. Aquí se define la lógica, los datos (propiedades) y las funciones (métodos) que el componente utilizará. El decorador `@Component` es crucial, ya que contiene la metadata que le dice a Angular cómo tratar a esta clase.

  - `selector`: La etiqueta HTML que usarás para insertar el componente en el template de otro componente (ej. `<app-perfil-usuario`>).

  - `templateUrl`: La ruta al archivo HTML del componente.

  - `styleUrls`: Un arreglo con las rutas a los archivos de estilos del componente.

- `perfil-usuario.component.html` (La Vista): El template que contiene el HTML del componente. Aquí es donde se define la estructura de la interfaz de usuario que este componente va a mostrar. Puedes usar la sintaxis de Angular para mostrar datos o manejar eventos.

- `perfil-usuario.component.css` (Los Estilos): Contiene los estilos CSS específicos para este componente. Es importante destacar que estos estilos están encapsulados, lo que significa que solo afectarán a los elementos dentro de este componente y no a otros en la aplicación.

- `perfil-usuario.component.spec.ts` (Las Pruebas): Este archivo se usa para escribir las pruebas unitarias del componente.
