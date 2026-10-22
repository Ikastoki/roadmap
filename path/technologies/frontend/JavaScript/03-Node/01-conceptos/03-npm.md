<h1 align="center">NPM</h1>

<h2>📑 Contenido</h2>

- [¿Qué es NPM?](#qué-es-npm)
- [package.json](#packagejson)
- [Gestión de Paquetes (CRUD)](#gestión-de-paquetes-crud)
  - [Crear (Instalar)](#crear-instalar)
  - [Read (Listar)](#read-listar)
  - [Update (Actualizar)](#update-actualizar)
  - [Delete (Desinstalar)](#delete-desinstalar)
- [Scripts](#scripts)
- [¿Qué es npx?](#qué-es-npx)
  - [¿Cómo funciona npx?](#cómo-funciona-npx)
  - [Diferencia entre npm y npx](#diferencia-entre-npm-y-npx)

## ¿Qué es NPM?

NPM es el gestor de paquetes por defecto de Node.js y la biblioteca de software más grande del mundo. Es una herramienta de línea de comandos que te permite:

- Instalar y gestionar librerías, paquetes y dependencias en tu proyecto.

- Compartir tu propio código con otros desarrolladores.

- Definir scripts para automatizar tareas en tu proyecto.

Cuando instalas Node.js, NPM viene incluido por defecto, así que no necesitas instalarlo por separado.

## package.json

Este archivo es el manifiesto de tu proyecto. Se encuentra en la raíz de tu proyecto y contiene metadatos cruciales, como el nombre, la versión, la descripción y, lo más importante, las dependencias de tu proyecto.

Puedes crear un archivo package.json de forma interactiva ejecutando:

```bash
npm init

# O de forma rápida con los valores por defecto:
npm init -y
```

## Gestión de Paquetes (CRUD)

El acrónimo CRUD (Crear, Leer, Actualizar, Eliminar) se aplica perfectamente a la gestión de paquetes con NPM.

### Crear (Instalar)

Para instalar un nuevo paquete y agregarlo a tu proyecto, usas el comando `npm install`.

Instalar un paquete localmente: Esto lo agrega como una dependencia en tu package.json.

```bash
# Instalar un paquete localmente: Esto lo agrega como una dependencia en tu package.json.
npm install nombre-del-paquete

# Instalar una dependencia de desarrollo: Son paquetes que solo necesitas durante el desarrollo (ej. linters, transpiladores).
npm install nombre-del-paquete --save-dev

# Instalar todos los paquetes de un proyecto: Cuando clonas un proyecto, solo necesitas ejecutar:
npm install
```

### Read (Listar)

Para ver los paquetes instalados en tu proyecto, puedes usar:

```bash
npm list
```

### Update (Actualizar)

Para actualizar un paquete a su última versión compatible (basada en la semántica del control de versiones), usas:

```bash
npm update <nombre-del-paquete>
```

### Delete (Desinstalar)

Para desinstalar un paquete y eliminarlo de tu package.json, usas:

```bash
npm uninstall <nombre-del-paquete>
```

## Scripts

La sección "`scripts`" en tu package.json te permite definir atajos de línea de comandos para automatizar tareas. Es una práctica común y muy útil.

Ejemplo de package.json con scripts:

```json
{
  "name": "mi-proyecto",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^2.0.20"
  }
}
```

- `npm start`: Es un comando estándar para iniciar tu aplicación. Aquí, se ejecutará el archivo index.js.

- `npm test`: Otro comando estándar para ejecutar tus pruebas.

- `npm run dev`: Para ejecutar cualquier otro script personalizado, usas `npm run nombre-del-script`. En este caso, `npm run dev` iniciaría la aplicación con `nodemon`, una herramienta que reinicia automáticamente el servidor cuando detecta cambios en los archivos.

## ¿Qué es npx?

`npx` es una herramienta de línea de comandos que viene incluida con NPM desde la versión 5.2. Su nombre significa "Node Package Execute".

Su propósito principal es ejecutar paquetes de Node.js sin tener que instalarlos globalmente en tu sistema o localmente en tu proyecto. Esto es extremadamente útil para las siguientes situaciones:

1. Ejecutar herramientas de una sola vez: Si necesitas usar una herramienta una sola vez (por ejemplo, para inicializar un proyecto), no tiene sentido instalarla globalmente. `npx` la descarga temporalmente, la ejecuta y la elimina.

2. Usar versiones específicas: Te permite ejecutar una versión de un paquete sin tener que lidiar con problemas de compatibilidad o conflictos entre versiones.

### ¿Cómo funciona npx?

Cuando ejecutas `npx nombre-del-paquete`, `npx` hace lo siguiente:

- Primero, verifica si el paquete ya está instalado en tu proyecto local (en la carpeta `node_modules`).

- Si no lo encuentra, lo descarga temporalmente en una caché global.

- Ejecuta el comando asociado al paquete.

- Limpia la caché de la descarga temporal (si es un paquete de un solo uso).

### Diferencia entre npm y npx

| Característica    | npm                                    | npx                                                           |
| ----------------- | -------------------------------------- | ------------------------------------------------------------- |
| Función principal | Instala y gestiona paquetes            | Ejecuta paquetes                                              |
| Uso               | `npm install paquete`                  | `npx paquete`                                                 |
| Almacenamiento    | Instala los paquetes en `node_modules` | Descarga y ejecuta temporalmente, o usa los de `node_modules` |
| Casos de uso      | Añadir dependencias al proyecto        | Ejecutar comandos de forma rápida                             |
