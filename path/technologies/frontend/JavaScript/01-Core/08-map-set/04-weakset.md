<h1 align='center'>WeakSet</h1>

<h2>📑 Contenido</h2>

- [WeakSet](#weakset)
- [Características de WeakSet](#características-de-weakset)
- [Creación de un WeakSet](#creación-de-un-weakset)
- [Métodos de WeakSet](#métodos-de-weakset)
  - [add(objeto)](#addobjeto)
  - [has(objeto)](#hasobjeto)
  - [delete(objeto)](#deleteobjeto)
- [Uso típico de WeakSet](#uso-típico-de-weakset)
- [Cuándo usar Set y WeakSet](#cuándo-usar-set-y-weakset)
  - [Usa Set cuando:](#usa-set-cuando)
  - [Usa WeakSet cuando:](#usa-weakset-cuando)

## WeakSet

Un WeakSet es similar a un Set, pero con diferencias importantes en el manejo de sus valores. Al igual que un Set, un WeakSet almacena valores únicos, pero solo permite almacenar objetos (no valores primitivos como números o cadenas), y las referencias a los objetos dentro del WeakSet son débiles. Esto significa que si no hay otras referencias a un objeto que está en un WeakSet, este puede ser eliminado por el recolector de basura.

## Características de WeakSet

Solo puede almacenar objetos como valores, no acepta tipos de datos primitivos.
Los objetos en un WeakSet tienen referencias débiles, lo que significa que si un objeto ya no tiene otras referencias en el programa, puede ser eliminado automáticamente.

- No es iterable: No se puede recorrer un WeakSet, y no se puede obtener su tamaño.
- No admite métodos como `clear()` ni tiene la propiedad `size`.

## Creación de un WeakSet

Un WeakSet se crea de forma similar a un Set, pero solo acepta objetos como valores.

```js
let weakSet = new WeakSet();
let obj1 = { nombre: "Juan" };
let obj2 = { nombre: "Ana" };

weakSet.add(obj1);
weakSet.add(obj2);
```

## Métodos de WeakSet

### add(objeto)

Añade un objeto al WeakSet.

```js
weakSet.add({ nombre: "Pedro" });
```

### has(objeto)

Verifica si un objeto está presente en el WeakSet.

```js
console.log(weakSet.has(obj1)); // true
```

### delete(objeto)

Elimina un objeto del WeakSet.

```js
weakSet.delete(obj2);
```

## Uso típico de WeakSet

Un caso común para usar WeakSet es cuando se necesita realizar un seguimiento de objetos sin evitar que estos sean recolectados por el recolector de basura si ya no son necesarios en otras partes del programa. Esto es útil en situaciones donde los objetos son temporales o pueden ser eliminados sin afectar la aplicación.

Por ejemplo, puedes usar WeakSet para marcar objetos temporalmente sin preocuparte por la gestión manual de la memoria:

```js
let ws = new WeakSet();
let elemento = document.querySelector(".elemento");
ws.add(elemento);

// Si 'elemento' es eliminado del DOM y no tiene más referencias, será recolectado por el recolector de basura.
elemento = null; // El objeto puede ser eliminado automáticamente del WeakSet
```

## Cuándo usar Set y WeakSet

### Usa Set cuando:

- Necesites almacenar valores únicos de cualquier tipo (números, cadenas, objetos, etc.).
- Quieras iterar sobre los valores o necesites saber cuántos elementos hay.
- Estés buscando eliminar duplicados de una lista.

### Usa WeakSet cuando:

- Necesites almacenar objetos de manera temporal sin afectar la gestión de memoria.
- Quieras asegurarte de que los objetos se eliminen cuando ya no se necesiten en el resto de la aplicación.
- No necesites iterar sobre los valores ni conocer el tamaño de la colección.
