<h1 align="center">Paquetes de Terceros</h1>

<h2>📑 Contenido</h2>

- [¿Qué son los Paquetes de Terceros?](#qué-son-los-paquetes-de-terceros)
- [¿Cómo se usan?](#cómo-se-usan)
  - [Instalación](#instalación)
  - [Importación y Uso](#importación-y-uso)

## ¿Qué son los Paquetes de Terceros?

Los paquetes de terceros son librerías y módulos de código que no forman parte de los módulos nativos de Node.js. Son creados por la comunidad de desarrolladores y se publican en el registro de NPM para que cualquiera pueda usarlos.

A diferencia de los módulos nativos, que ya están incluidos en Node.js, los paquetes de terceros deben ser instalados en tu proyecto para poder ser usados. El ecosistema de NPM es tan vasto que hay un paquete para casi cualquier tarea que puedas imaginar: desde la creación de servidores hasta la manipulación de fechas, la validación de datos o la conexión con bases de datos.

## ¿Cómo se usan?

El proceso es simple y consta de dos pasos principales:

### Instalación

Para instalar un paquete, usas el comando `npm install` en la terminal, dentro del directorio de tu proyecto. Esto descarga el paquete del registro de NPM y lo guarda en la carpeta node_modules de tu proyecto. Además, se registra en el archivo package.json para que otros desarrolladores puedan instalar las mismas dependencias.

### Importación y Uso

Una vez que el paquete está instalado, lo importas en tu código de la misma manera que lo harías con un módulo nativo, usando `require()` (CommonJS) o `import (ESM)`.
