<h1 align="center">Configuración y Estructura </h1>

<h2>📑 Contenido</h2>

- [Estructura de archivos y carpetas](#estructura-de-archivos-y-carpetas)
- [¡Claro! Hablemos de la configuración y la estructura de un proyecto de Node.js. Una buena organización es clave para la escalabilidad y el mantenimiento.](#claro-hablemos-de-la-configuración-y-la-estructura-de-un-proyecto-de-nodejs-una-buena-organización-es-clave-para-la-escalabilidad-y-el-mantenimiento)
- [Archivos de Configuración](#archivos-de-configuración)
  - [package.json](#packagejson)
  - [Variables de entorno (.env)](#variables-de-entorno-env)
- [Archivos de configuración personalizados](#archivos-de-configuración-personalizados)

## Estructura de archivos y carpetas

Aunque Node.js no impone una estructura específica, existen convenciones comunes que facilitan la vida del desarrollador. Una estructura típica para una aplicación web con Express se vería así:

```
/mi-proyecto
|-- node_modules/         # Donde se instalan las dependencias de NPM
|-- public/               # Archivos públicos que el navegador puede acceder (HTML, CSS, JS)
|   |-- css/
|   |-- js/
|   |-- index.html
|-- src/                  # Código fuente de tu aplicación (opcional, para proyectos grandes)
|   |-- controllers/      # Lógica de la aplicación
|   |-- models/           # Lógica para interactuar con la base de datos
|   |-- routes/           # Definición de las rutas de tu API
|   |-- services/         # Lógica de negocio
|   |-- app.js            # Archivo principal de la aplicación
|-- .env                  # Archivo para variables de entorno
|-- .gitignore            # Archivo para ignorar archivos en Git
|-- package.json          # El manifiesto del proyecto
|-- package-lock.json     # Guarda la versión exacta de las dependencias
|-- README.md             # Información del proyecto
```

Esta estructura separa las responsabilidades en carpetas lógicas, lo que hace que el proyecto sea más modular y fácil de navegar.

## ¡Claro! Hablemos de la configuración y la estructura de un proyecto de Node.js. Una buena organización es clave para la escalabilidad y el mantenimiento.

Estructura de archivos y carpetas
Aunque Node.js no impone una estructura específica, existen convenciones comunes que facilitan la vida del desarrollador. Una estructura típica para una aplicación web con Express se vería así:

/mi-proyecto
|-- node_modules/ # Donde se instalan las dependencias de NPM
|-- public/ # Archivos públicos que el navegador puede acceder (HTML, CSS, JS)
| |-- css/
| |-- js/
| |-- index.html
|-- src/ # Código fuente de tu aplicación (opcional, para proyectos grandes)
| |-- controllers/ # Lógica de la aplicación
| |-- models/ # Lógica para interactuar con la base de datos
| |-- routes/ # Definición de las rutas de tu API
| |-- services/ # Lógica de negocio
| |-- app.js # Archivo principal de la aplicación
|-- .env # Archivo para variables de entorno
|-- .gitignore # Archivo para ignorar archivos en Git
|-- package.json # El manifiesto del proyecto
|-- package-lock.json # Guarda la versión exacta de las dependencias
|-- README.md # Información del proyecto
Esta estructura separa las responsabilidades en carpetas lógicas, lo que hace que el proyecto sea más modular y fácil de navegar.

## Archivos de Configuración

En Node.js, es una práctica estándar separar la configuración de la aplicación del código fuente.

### package.json

Ya lo discutimos, pero es el archivo de configuración más importante. Define el nombre del proyecto, la versión, los scripts de la línea de comandos y las dependencias.

### Variables de entorno (.env)

Las variables de entorno son un mecanismo para inyectar configuraciones externas a la aplicación, como claves de API, URLs de bases de datos o puertos de servidor. Se almacenan en un archivo .env en la raíz del proyecto.

Ventajas:

- Seguridad: Mantiene la información sensible (contraseñas, claves) fuera del código fuente, lo que evita que se suba a repositorios públicos como GitHub.

- Flexibilidad: Te permite cambiar la configuración (por ejemplo, de desarrollo a producción) sin modificar el código.

```toml
PUERTO=3000
DATABASE_URL=mongodb://localhost:27017/mi-aplicacion
CLAVE_API_SECRETA=abc123xyz
```

Para usar estas variables en tu código, necesitas un paquete como dotenv.

```bash
npm install dotenv
```

```javascript
// En tu archivo principal (ej. app.js)
require("dotenv").config();

const puerto = process.env.PUERTO; // Accedes a la variable de entorno
console.log(`Servidor escuchando en el puerto ${puerto}`);
```

## Archivos de configuración personalizados

Para configuraciones más complejas, puedes crear archivos config.js o carpetas de configuración para gestionar diferentes entornos (desarrollo, pruebas, producción).

```javascript
// config/config.js
module.exports = {
  desarrollo: {
    puerto: 3000,
    baseDatos: "mongodb://localhost:27017/dev",
  },
  produccion: {
    puerto: process.env.PUERTO,
    baseDatos: process.env.DATABASE_URL,
  },
};
```

Luego, en tu aplicación, puedes cargar la configuración apropiada según la variable de entorno `NODE_ENV`.

```javascript
const env = process.env.NODE_ENV || "desarrollo";
const config = require("./config/config")[env];
```
