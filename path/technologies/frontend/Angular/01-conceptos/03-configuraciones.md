<h1 align="center">Configuraciones</h1>

<h2>📑 Contenido</h2>

- [Instalar Node.js y npm](#instalar-nodejs-y-npm)
- [Instalar Angular CLI](#instalar-angular-cli)
- [Crear un nuevo proyecto](#crear-un-nuevo-proyecto)
- [Ejecutar la aplicación](#ejecutar-la-aplicación)

## Instalar Node.js y npm

Angular se basa en Node.js y su gestor de paquetes, npm. Si aún no los tienes, debes instalarlos. Angular requiere una versión LTS activa o de mantenimiento de Node.js. Puedes descargarlos desde la página oficial de Node.js.

Para verificar que la instalación fue exitosa, abre tu terminal y ejecuta estos comandos:

```bash
node -v
npm -v
```

Si ambos comandos te devuelven un número de versión, estás listo para el siguiente paso.

## Instalar Angular CLI

El Angular CLI (Command Line Interface) es la herramienta de línea de comandos oficial que simplifica y agiliza el desarrollo con Angular. Te permite crear proyectos, generar componentes, servicios, módulos y mucho más.

Para instalar la CLI globalmente en tu sistema, usa el siguiente comando:

```bash
npm install -g @angular/cli
```

El flag `-g` significa que la herramienta se instalará de forma global, lo que te permitirá usar el comando `ng` en cualquier directorio de tu sistema.

## Crear un nuevo proyecto

Una vez que la CLI está instalada, puedes crear tu primer proyecto de Angular. Navega hasta la carpeta donde deseas guardarlo y ejecuta el siguiente comando:

```bash
ng new mi-primer-app
```

Cuando ejecutes el comando, la CLI te hará algunas preguntas sobre las características que deseas incluir en tu proyecto, como las rutas de navegación y el formato de los estilos (CSS, SCSS, etc.).

La CLI creará una nueva carpeta llamada **mi-primer-app** con todos los archivos y dependencias necesarios para que tu aplicación funcione.

## Ejecutar la aplicación

Una vez que la CLI termine de crear el proyecto, navega a la nueva carpeta y ejecuta el servidor de desarrollo.

```bash
cd mi-primer-app
ng serve --open
```

El comando `ng serve` compila la aplicación y lanza un servidor de desarrollo local, mientras que el `flag --open` abrirá automáticamente tu navegador en la dirección `http://localhost:4200/`, donde podrás ver tu aplicación en funcionamiento.
