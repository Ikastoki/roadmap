<h1 align="center">Rutas</h1>

<h2>📑 Contenido</h2>

- [Módulo path (Rutas)](#módulo-path-rutas)
- [Métodos más importantes](#métodos-más-importantes)
  - [path.join()](#pathjoin)
  - [path.resolve()](#pathresolve)
  - [path.basename()](#pathbasename)
  - [path.extname()](#pathextname)
  - [path.dirname()](#pathdirname)

## Módulo path (Rutas)

El módulo path es un módulo nativo de Node.js que te permite trabajar con rutas de archivos y directorios de una manera predecible y multiplataforma. Es crucial porque las rutas de archivos son diferentes entre sistemas operativos (Windows usa \ y sistemas Unix como Linux y macOS usan /). Este módulo se encarga de que tu código funcione correctamente sin importar el sistema operativo en el que se ejecute.

Para usarlo, lo importas de la siguiente manera:

```javascript
const path = require("node:path");
```

## Métodos más importantes

### path.join()

Este método es uno de los más utilizados. Junta todos los segmentos de una ruta que se le pasen como argumentos. Se encarga de usar el separador correcto (`\` o `/`) para el sistema operativo en el que se ejecuta, evitando así errores.

```javascript
const rutaUnida = path.join("/usr", "local", "bin", "ejemplo.js");
console.log(rutaUnida); // Salida en Unix: /usr/local/bin/ejemplo.js
```

### path.resolve()

Te permite obtener la ruta absoluta de un archivo. Si no se le pasa ningún argumento, devuelve la ruta del directorio de trabajo actual. Si se le pasan segmentos, los une y resuelve la ruta absoluta.

```javascript
const rutaAbsoluta = path.resolve("directorio", "archivo.txt");
console.log(rutaAbsoluta); // Salida: /Users/usuario/proyecto/directorio/archivo.txt
```

### path.basename()

Devuelve la parte final de una ruta, que suele ser el nombre del archivo. También puedes pasarle una extensión para que la omita.

```javascript
const nombreArchivo = path.basename("/home/user/documento.txt");
console.log(nombreArchivo); // Salida: documento.txt

const nombreSinExt = path.basename("/home/user/documento.txt", ".txt");
console.log(nombreSinExt); // Salida: documento
```

### path.extname()

Devuelve la extensión de un archivo.

```javascript
const extension = path.extname("miarchivo.html");
console.log(extension); // Salida: .html
```

### path.dirname()

Devuelve el nombre del directorio de una ruta, eliminando la parte del archivo.

```javascript
const directorio = path.dirname("/home/user/documento.txt");
console.log(directorio); // Salida: /home/user
```

Usar el módulo ''path'' es una buena práctica porque hace que tu código sea portable. En lugar de concatenar cadenas con barras ('/'), utilizas estos métodos para que Node.js maneje la lógica por ti.
