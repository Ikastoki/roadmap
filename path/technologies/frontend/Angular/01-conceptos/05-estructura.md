<h1 align="center">Estructura del proyecto</h1>

<h2>📑 Contenido</h2>

- [Estructura del proyecto](#estructura-del-proyecto)
- [Archivos y Carpetas Principales](#archivos-y-carpetas-principales)
- [La Carpeta src/ (El corazón de la aplicación)](#la-carpeta-src-el-corazón-de-la-aplicación)

## Estructura del proyecto

La estructura de un proyecto de Angular, generada por el Angular CLI, está pensada para ser organizada y escalable.

## Archivos y Carpetas Principales

```
my-app/
├── e2e/
├── node_modules/
├── src/
├── angular.json
├── package.json
├── package-lock.json
├── tsconfig.json
├── README.md
```

- `src/`: Esta es la carpeta más importante, ya que contiene todo el código fuente de tu aplicación. Aquí es donde pasarás la mayor parte del tiempo desarrollando.

- `node_modules/`: Contiene todas las librerías y dependencias de JavaScript que tu proyecto necesita. Esta carpeta se gestiona automáticamente con npm y no debes modificarla directamente.

- `angular.json`: El archivo de configuración del Angular CLI. Aquí se definen las configuraciones de construcción, despliegue, pruebas, y qué archivos se consideran parte de tu aplicación.

- `package.json`: Es el manifiesto de tu proyecto. Contiene metadatos, los scripts que puedes ejecutar (ng serve, ng build, etc.) y la lista de dependencias de tu proyecto (@angular/core, rxjs, etc.).

- `tsconfig.json`: El archivo de configuración para el compilador de TypeScript. Aquí puedes definir las reglas y opciones para compilar tu código.

## La Carpeta src/ (El corazón de la aplicación)

Dentro de `src/`, la estructura es la siguiente:

```
src/
├── app/
├── assets/
├── environments/
├── favicon.ico
├── index.html
├── main.ts
├── styles.css
```

- `app/`: Es el directorio principal para el código de tu aplicación. Aquí es donde se crean los componentes, servicios, módulos y otros elementos lógicos.

- `assets/`: Un lugar para almacenar recursos estáticos como imágenes, iconos, fuentes y otros archivos que no son código de Angular.

- `environments/`: Contiene archivos de configuración específicos para diferentes entornos (por ejemplo, environment.ts para desarrollo y environment.prod.ts para producción).

- `favicon.ico`: El icono que se muestra en la pestaña del navegador.

- `index.html`: El único archivo HTML de la aplicación. Angular es una Single Page Application (SPA), lo que significa que todo el contenido se carga y se inserta dinámicamente en este archivo.

- `main.ts`: El punto de entrada principal de la aplicación. Aquí es donde Angular inicia o arranca el módulo raíz de tu aplicación (AppModule).

- `styles.css`: Un archivo para estilos globales. Cualquier estilo que definas aquí afectará a todos los componentes de tu aplicación.
