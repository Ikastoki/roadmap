<h1 align='center'>Index DB</h1>

<h2>📑 Contenido</h2>

- [Index DB](#index-db)
- [¿Qué es IndexedDB?](#qué-es-indexeddb)
  - [Las principales características de IndexedDB son:](#las-principales-características-de-indexeddb-son)
- [Estructura básica de IndexedDB](#estructura-básica-de-indexeddb)
- [Primeros pasos con IndexedDB](#primeros-pasos-con-indexeddb)
- [Operaciones básicas con IndexedDB](#operaciones-básicas-con-indexeddb)
  - [Agregar datos a una object store:](#agregar-datos-a-una-object-store)
  - [Leer datos de una object store:](#leer-datos-de-una-object-store)
  - [Actualizar un registro existente:](#actualizar-un-registro-existente)
  - [Eliminar un registro:](#eliminar-un-registro)
- [Uso de índices para búsquedas eficientes](#uso-de-índices-para-búsquedas-eficientes)
- [Transacciones en IndexedDB](#transacciones-en-indexeddb)
  - [Las transacciones tienen tres modos:](#las-transacciones-tienen-tres-modos)
- [Manejo de errores](#manejo-de-errores)

## Index DB

IndexedDB es una API de almacenamiento de datos clave-valor en el lado del cliente, diseñada para aplicaciones web que necesitan almacenar grandes cantidades de datos estructurados. A diferencia de otras opciones como `localStorage`, IndexedDB es una base de datos transaccional orientada a objetos que permite almacenar datos de forma persistente, realizar consultas, búsquedas y manipulación de registros, e incluso trabajar con datos binarios.

IndexedDB es particularmente útil cuando se necesita almacenar grandes cantidades de datos o cuando los datos necesitan ser accesibles sin conexión. Su diseño permite a las aplicaciones web ofrecer una experiencia más fluida, sin depender completamente del servidor.

## ¿Qué es IndexedDB?

IndexedDB es una base de datos del lado del cliente, orientada a objetos, donde los datos se almacenan en colecciones llamadas object stores. A diferencia de bases de datos relacionales, IndexedDB no utiliza SQL, sino un modelo basado en claves y valores. Cada valor almacenado en una object store está asociado con una clave, y estas claves son únicas dentro de esa tienda.

### Las principales características de IndexedDB son:

- **Transacciones:** Todas las operaciones en IndexedDB (lectura, escritura, actualización, eliminación) se realizan dentro de transacciones, lo que garantiza consistencia y permite revertir cambios en caso de errores.
- **Datos estructurados:** IndexedDB permite almacenar datos complejos como objetos JavaScript, arrays e incluso blobs (archivos binarios).
- **Asincronía:** IndexedDB utiliza un enfoque basado en eventos y callbacks o promesas para manejar operaciones, lo que previene bloqueos en la interfaz de usuario.
- **Soporte offline:** Los datos almacenados en IndexedDB son persistentes y están disponibles incluso cuando el usuario no tiene conexión a internet.

## Estructura básica de IndexedDB

IndexedDB utiliza una estructura de base de datos jerárquica:

- **Base de datos (Database):** Es el contenedor principal de la información. Cada base de datos tiene un nombre y una versión.
- **Object Store:** Dentro de cada base de datos, los datos se organizan en object stores, que son el equivalente a las tablas en bases de datos relacionales. Cada object store contiene múltiples registros, que son pares clave-valor.
- **Índices (Indexes):** Permiten realizar consultas eficientes dentro de una object store, proporcionando acceso a registros basados en valores de propiedades específicas.
- **Clave:** Cada registro dentro de una object store debe tener una clave única que lo identifique.

## Primeros pasos con IndexedDB

Para empezar a trabajar con IndexedDB, es necesario abrir una conexión a la base de datos. Esto se hace utilizando el método `indexedDB.open()`, que acepta dos parámetros: el nombre de la base de datos y su versión.

```js
let db;
let request = indexedDB.open("MiBaseDeDatos", 1);

request.onerror = function (event) {
  console.log("Error al abrir la base de datos", event);
};

request.onsuccess = function (event) {
  db = request.result;
  console.log("Base de datos abierta con éxito");
};

request.onupgradeneeded = function (event) {
  db = event.target.result;

  // Crear una object store (si no existe)
  let objectStore = db.createObjectStore("productos", { keyPath: "id" });

  // Crear índices (opcional)
  objectStore.createIndex("nombre", "nombre", { unique: false });
  objectStore.createIndex("precio", "precio", { unique: false });

  console.log("Object store y índices creados");
};
```

Descripción del código:

- Se intenta abrir la base de datos MiBaseDeDatos. Si no existe, se crea.
- El segundo parámetro (1) es la versión de la base de datos. Cada vez que se quiera realizar un cambio en la estructura (añadir una nueva object store, cambiar índices, etc.), se debe incrementar la versión.
- Si se crea una nueva base de datos (o se actualiza su versión), se ejecuta el evento onupgradeneeded, donde se definen las object stores e índices.
- En este ejemplo, se crea una object store llamada "productos", donde cada registro estará identificado por el campo id. También se crean índices en los campos nombre y precio.

## Operaciones básicas con IndexedDB

Una vez que la base de datos está abierta, se pueden realizar diversas operaciones como agregar, leer, actualizar o eliminar datos. Todas estas operaciones se realizan dentro de transacciones.

### Agregar datos a una object store:

```js
let transaction = db.transaction(["productos"], "readwrite");
let objectStore = transaction.objectStore("productos");

let request = objectStore.add({ id: 1, nombre: "Laptop", precio: 1000 });

request.onsuccess = function (event) {
  console.log("Producto agregado con éxito");
};

request.onerror = function (event) {
  console.log("Error al agregar el producto", event);
};
```

En este ejemplo, se crea una transacción en modo readwrite para la object store "productos" y se utiliza el método `add()` para agregar un nuevo objeto (producto).

### Leer datos de una object store:

```js
let transaction = db.transaction(["productos"]);
let objectStore = transaction.objectStore("productos");

let request = objectStore.get(1);

request.onsuccess = function (event) {
  if (request.result) {
    console.log("Producto encontrado:", request.result);
  } else {
    console.log("Producto no encontrado");
  }
};

request.onerror = function (event) {
  console.log("Error al leer el producto", event);
};
```

En este caso, se utiliza el método `get()` para obtener un producto por su clave (en este caso, id: 1).

### Actualizar un registro existente:

```js
let transaction = db.transaction(["productos"], "readwrite");
let objectStore = transaction.objectStore("productos");

let request = objectStore.get(1);

request.onsuccess = function (event) {
  let producto = request.result;
  producto.precio = 1200;

  let updateRequest = objectStore.put(producto);

  updateRequest.onsuccess = function (event) {
    console.log("Producto actualizado con éxito");
  };
};
```

Aquí, primero se obtiene el registro que se desea modificar usando `get()`, luego se actualiza el valor y se guarda de nuevo con `put()`.

### Eliminar un registro:

```js
let transaction = db.transaction(["productos"], "readwrite");
let objectStore = transaction.objectStore("productos");

let request = objectStore.delete(1);

request.onsuccess = function (event) {
  console.log("Producto eliminado con éxito");
};
```

El método `delete()` elimina el registro asociado con la clave proporcionada, en este caso id: 1.

## Uso de índices para búsquedas eficientes

Los índices permiten realizar búsquedas basadas en propiedades que no son la clave principal. Se definen en el momento de crear o actualizar la estructura de la base de datos (onupgradeneeded).

```js
// Buscar productos por nombre:
let transaction = db.transaction(["productos"]);
let objectStore = transaction.objectStore("productos");
let index = objectStore.index("nombre");

let request = index.get("Laptop");

request.onsuccess = function (event) {
  if (request.result) {
    console.log("Producto encontrado:", request.result);
  } else {
    console.log("Producto no encontrado");
  }
};
```

En este ejemplo, se busca un producto por el valor del campo nombre utilizando el índice nombre.

## Transacciones en IndexedDB

Todas las operaciones en IndexedDB se realizan dentro de transacciones. Una transacción agrupa varias operaciones de lectura o escritura, y garantiza que todas se ejecuten de manera atómica, es decir, que o bien todas se completan con éxito o ninguna se ejecuta.

### Las transacciones tienen tres modos:

- **readonly:** Solo permite leer datos.
- **readwrite:** Permite leer y escribir datos.
- **versionchange:** Se utiliza para cambiar la estructura de la base de datos (por ejemplo, añadir nuevas object stores o índices).

## Manejo de errores

Es importante manejar los errores en IndexedDB, especialmente porque las operaciones son asincrónicas. Cada operación devuelve una solicitud (request), y es necesario implementar manejadores de eventos para controlar tanto los éxitos como los errores.

```js
let request = indexedDB.open("MiBaseDeDatos", 1);

request.onerror = function (event) {
  console.error("Error al abrir la base de datos:", event);
};

request.onsuccess = function (event) {
  let db = event.target.result;
  console.log("Base de datos abierta con éxito");
};
```
