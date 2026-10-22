<h1 align='center'>WeakMap</h1>

<h2>📑 Contenido</h2>

- [WeakMap](#weakmap)
- [Características de WeakMap](#características-de-weakmap)
- [Creación de un WeakMap](#creación-de-un-weakmap)
- [Métodos de WeakMap](#métodos-de-weakmap)
  - [set(clave, valor)](#setclave-valor)
  - [get(clave)](#getclave)
  - [has(clave)](#hasclave)
  - [delete(clave)](#deleteclave)
- [Uso típico de WeakMap](#uso-típico-de-weakmap)
- [Cuándo usar Map y WeakMap](#cuándo-usar-map-y-weakmap)
  - [Usa Map cuando:](#usa-map-cuando)
  - [Usa WeakMap cuando\_](#usa-weakmap-cuando_)

## WeakMap

Un WeakMap es similar a un Map, pero con diferencias clave que lo hacen adecuado para casos de uso específicos. Los pares clave-valor en un WeakMap tienen referencias débiles. Esto significa que si no hay otras referencias a un objeto clave, este puede ser recogido por el recolector de basura de JavaScript, liberando la memoria automáticamente. En otras palabras, los WeakMap permiten almacenar objetos como claves sin evitar que estos objetos sean eliminados cuando ya no se usan en ningún otro lugar del programa.

## Características de WeakMap

- Las claves deben ser objetos. No se pueden usar valores primitivos (números, cadenas, booleanos, etc.) como claves.
- Los objetos almacenados como claves pueden ser recogidos por el recolector de basura si no hay otras referencias a ellos.
- No es iterable: no puedes recorrer un WeakMap ni obtener su tamaño, ya que las claves pueden ser eliminadas automáticamente en cualquier momento.
- Está diseñado para situaciones en las que necesitas asociar datos temporales a objetos, pero no quieres que esos datos impidan que el objeto sea eliminado cuando ya no es necesario.

## Creación de un WeakMap

Un WeakMap se crea de forma similar a un Map, pero solo puede aceptar objetos como claves.

```js
let weakMap = new WeakMap();
let obj = { nombre: "Juan" };
weakMap.set(obj, "algún valor");
```

## Métodos de WeakMap

### set(clave, valor)

Añade o actualiza un par clave-valor en el WeakMap.

```js
let obj = { nombre: "Ana" };
weakMap.set(obj, "valor asociado");
```

### get(clave)

Devuelve el valor asociado con la clave dada.

```js
console.log(weakMap.get(obj)); // 'valor asociado'
```

### has(clave)

Verifica si una clave está presente en el WeakMap.

```js
console.log(weakMap.has(obj)); // true
```

### delete(clave)

Elimina el par clave-valor asociado con la clave dada.

```js
weakMap.delete(obj);
```

## Uso típico de WeakMap

Un caso típico de uso de WeakMap es cuando necesitas asociar metadatos o información auxiliar a un objeto, pero sin impedir que ese objeto sea eliminado por el recolector de basura cuando ya no se usa.

```js
let wm = new WeakMap();
let elemento = document.querySelector(".elemento");

// Asociar datos a un elemento del DOM
wm.set(elemento, { información: "datos extra" });

// Eliminar la referencia cuando ya no se necesita
elemento = null; // El objeto puede ser recolectado por el recolector de basura
```

En este ejemplo, si el elemento DOM (elemento) es eliminado de la página o ya no es necesario, el recolector de basura puede liberarlo y los datos asociados en el WeakMap también serán liberados automáticamente.

## Cuándo usar Map y WeakMap

### Usa Map cuando:

- Necesites trabajar con claves de cualquier tipo.
- Quieras iterar sobre las entradas de la colección.
- Necesites controlar el tamaño de la colección.

### Usa WeakMap cuando\_

- Solo quieras usar objetos como claves.
- Quieras que los objetos sean eliminados automáticamente cuando ya no se necesiten (ideal para gestionar memoria).
- No te importe la iteración o conocer el tamaño de la colección.
- Estés trabajando con estructuras temporales donde no necesites almacenar datos de forma persistente.
