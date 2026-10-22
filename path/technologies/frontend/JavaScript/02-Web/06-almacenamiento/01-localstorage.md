<h1 align='center'>Local Storage</h1>

<h2>📑 Contenido</h2>

- [Local Storage](#local-storage)
- [¿Qué es Local Storage?](#qué-es-local-storage)
- [Usos Comunes de Local Storage](#usos-comunes-de-local-storage)
- [Funciones Principales de Local Storage](#funciones-principales-de-local-storage)
  - [setItem(): Guardar un valor](#setitem-guardar-un-valor)
  - [getItem(): Recuperar un valor](#getitem-recuperar-un-valor)
  - [removeItem(): Eliminar un valor](#removeitem-eliminar-un-valor)
  - [clear(): Vaciar el Local Storage](#clear-vaciar-el-local-storage)
  - [length y key(): Iterar sobre los elementos almacenados](#length-y-key-iterar-sobre-los-elementos-almacenados)
- [Ejemplo de Uso Completo](#ejemplo-de-uso-completo)
- [Limitaciones de Local Storage](#limitaciones-de-local-storage)

## Local Storage

Local Storage es una API de almacenamiento web que permite a las aplicaciones web almacenar datos en el navegador del usuario de manera persistente. Es parte del estándar de HTML5 y proporciona una forma simple y eficaz para guardar información localmente en el navegador, sin necesidad de utilizar cookies o bases de datos más complejas. Los datos almacenados en Local Storage no se eliminan al cerrar el navegador, lo que lo convierte en una opción ideal para guardar datos que deben persistir entre sesiones.

En este capítulo, exploraremos qué es Local Storage, cómo funciona y cómo puedes utilizarlo en tus proyectos JavaScript.

## ¿Qué es Local Storage?

Local Storage es un almacenamiento web que permite guardar pares clave-valor en el navegador del usuario. Tiene las siguientes características:

- **Persistencia:** Los datos almacenados no tienen fecha de caducidad. Permanecen almacenados incluso si el usuario cierra el navegador.
- **Tamaño:** El espacio disponible varía según el navegador, pero típicamente es de 5 a 10 MB por origen (por dominio).
- **Almacenamiento basado en clave-valor:** Los datos se guardan como pares clave-valor, donde ambas partes son cadenas de texto.

A diferencia de Session Storage, que solo persiste durante la sesión del navegador, Local Storage permite que los datos estén disponibles indefinidamente hasta que sean eliminados por el código o el usuario.

## Usos Comunes de Local Storage

Local Storage es útil en varios casos de uso, como:

- **Recordar preferencias del usuario:** Guardar configuraciones o temas seleccionados por el usuario.
- **Mantener el estado de una aplicación:** Almacenar el progreso de un usuario en una aplicación o formulario.
- **Datos temporales para aplicaciones sin conexión:** Almacenar información cuando la aplicación debe funcionar sin conexión, como listas de tareas o notas.
- **Guardar tokens de autenticación:** Guardar tokens de acceso que no se requieren en cada carga de página.

## Funciones Principales de Local Storage

Local Storage está disponible a través del objeto global localStorage y tiene varias funciones clave que permiten interactuar con los datos almacenados.

### setItem(): Guardar un valor

Este método permite almacenar un par clave-valor en Local Storage. Ambos deben ser cadenas de texto. Si el valor que deseas guardar es un objeto, necesitarás convertirlo a formato JSON.

```js
// Guardar un valor
localStorage.setItem("nombre", "Juan");

// Guardar un objeto como cadena JSON
const usuario = { nombre: "Juan", edad: 30 };
localStorage.setItem("usuario", JSON.stringify(usuario));
```

### getItem(): Recuperar un valor

Para recuperar un valor almacenado, usas `getItem()` pasando la clave como argumento. Si el valor almacenado es un objeto JSON, tendrás que convertirlo de nuevo a un objeto utilizando J`SON.parse()`.

```js
// Recuperar un valor simple
const nombre = localStorage.getItem("nombre");
console.log(nombre); // Juan

// Recuperar un objeto
const usuario = JSON.parse(localStorage.getItem("usuario"));
console.log(usuario.nombre); // Juan
```

### removeItem(): Eliminar un valor

Si deseas eliminar una clave y su valor asociado de Local Storage, puedes usar `removeItem()` pasando la clave.

```js
// Eliminar un valor
localStorage.removeItem("nombre");
```

### clear(): Vaciar el Local Storage

El método `clear()` borra todos los datos almacenados en Local Storage.

```js
// Limpiar todo el Local Storage
localStorage.clear();
```

### length y key(): Iterar sobre los elementos almacenados

Local Storage también permite conocer el número de elementos almacenados mediante `localStorage.length`, y puedes obtener las claves almacenadas usando `localStorage.key(index)`.

```js
// Obtener el número de elementos almacenados
const numeroElementos = localStorage.length;
console.log(numeroElementos);

// Obtener una clave específica por su índice
const primeraClave = localStorage.key(0);
console.log(primeraClave);
```

## Ejemplo de Uso Completo

Supongamos que estás desarrollando una aplicación donde los usuarios pueden configurar un tema oscuro o claro, y quieres que esa preferencia se guarde para la próxima vez que el usuario vuelva.

```js
// Guardar el tema preferido en Local Storage
function guardarTema(tema) {
  localStorage.setItem("tema", tema);
}

// Obtener el tema preferido desde Local Storage
function obtenerTema() {
  return localStorage.getItem("tema") || "claro"; // Valor predeterminado: 'claro'
}

// Aplicar el tema al cargar la página
function aplicarTema() {
  const tema = obtenerTema();
  document.body.className = tema;
}

// Cambiar tema y guardarlo en Local Storage
function cambiarTema(nuevoTema) {
  document.body.className = nuevoTema;
  guardarTema(nuevoTema);
}

// Evento para cambiar el tema cuando el usuario lo selecciona
document
  .getElementById("botonTemaOscuro")
  .addEventListener("click", function () {
    cambiarTema("oscuro");
  });

document
  .getElementById("botonTemaClaro")
  .addEventListener("click", function () {
    cambiarTema("claro");
  });

// Aplicar tema al cargar la página
aplicarTema();
```

Este ejemplo guarda la preferencia de tema del usuario en Local Storage, asegurando que la próxima vez que visite la página, el tema seleccionado se mantendrá.

## Limitaciones de Local Storage

Aunque Local Storage es una herramienta poderosa, también tiene algunas limitaciones que los desarrolladores deben tener en cuenta:

**Tamaño limitado:** Local Storage está limitado a unos 5-10 MB, dependiendo del navegador. No es adecuado para almacenar grandes cantidades de datos.

**Solo maneja cadenas de texto:** Local Storage solo puede almacenar cadenas. Si deseas almacenar objetos o arrays, debes convertirlos a JSON usando JSON.`stringify()`.

**Seguridad limitada:** Local Storage no es seguro para almacenar información sensible, como contraseñas o tokens de acceso críticos. Los datos almacenados son accesibles por cualquier script que se ejecute en la misma página, lo que lo hace vulnerable a ataques de tipo XSS (Cross-Site Scripting).

**Sin soporte para eventos de expiración:** Los datos almacenados en Local Storage no caducan automáticamente. Debes gestionarlo manualmente si necesitas una expiración.
