<h1 align='center'>Métodos Predefinidos</h1>

<h2>📑 Contenido</h2>

- [Métodos Predefinidos](#métodos-predefinidos)
- [Métodos Predefinidos de Objetos](#métodos-predefinidos-de-objetos)
  - [Object.keys()](#objectkeys)
  - [Object.values()](#objectvalues)
  - [Object.entries()](#objectentries)
  - [Object.assign()](#objectassign)
  - [Object.freeze()](#objectfreeze)
  - [Object.seal()](#objectseal)
  - [Object.getOwnPropertyDescriptors()](#objectgetownpropertydescriptors)
  - [Object.hasOwnProperty()](#objecthasownproperty)
  - [Object.create()](#objectcreate)
  - [Object.defineProperty()](#objectdefineproperty)
  - [Object.is()](#objectis)

## Métodos Predefinidos

Un método de un objeto es simplemente una función que se almacena como una propiedad dentro de un objeto. Estos métodos permiten que los objetos tengan comportamientos o acciones asociados con ellos. Los métodos pueden ser cualquier función, incluidas las funciones anónimas y las funciones flecha.

```js
const persona = {
  nombre: "Carlos",
  saludar: function () {
    console.log("Hola, mi nombre es " + this.nombre);
  },
};

persona.saludar(); // "Hola, mi nombre es Carlos"
```

En este ejemplo, saludar es un método de persona, y utiliza `this` para acceder a las propiedades del objeto.

## Métodos Predefinidos de Objetos

JavaScript proporciona varios métodos predefinidos para trabajar con objetos. Estos métodos están disponibles en el prototipo del objeto global `Object` y permiten realizar tareas comunes, como iterar sobre propiedades o copiar objetos.

### Object.keys()

Este método devuelve un array con los nombres de las propiedades enumerables de un objeto.

```js
const persona = { nombre: "Ana", edad: 30 };
const claves = Object.keys(persona);
console.log(claves); // ["nombre", "edad"]
```

### Object.values()

Devuelve un array con los valores de las propiedades enumerables del objeto.

```js
const valores = Object.values(persona);
console.log(valores); // ["Ana", 30]
```

### Object.entries()

Devuelve un array de arrays, donde cada sub-array es un par clave-valor del objeto.

```js
const entradas = Object.entries(persona);
console.log(entradas); // [["nombre", "Ana"], ["edad", 30]]
```

### Object.assign()

Copia todas las propiedades enumerables de uno o más objetos a un objeto destino. Retorna el objeto destino.

```js
const destino = {};
const origen = { a: 1, b: 2 };
Object.assign(destino, origen);
console.log(destino); // { a: 1, b: 2 }
```

### Object.freeze()

Congela un objeto, evitando que se puedan añadir, eliminar o modificar sus propiedades. Las propiedades existentes no se pueden cambiar, pero sus valores aún se pueden leer.

```js
const obj = { nombre: "Luis" };
Object.freeze(obj);

obj.nombre = "Carlos"; // No tiene efecto
console.log(obj.nombre); // "Luis"
```

### Object.seal()

Sella un objeto, evitando que se puedan añadir o eliminar propiedades. Sin embargo, las propiedades existentes se pueden modificar.

```js
const obj = { edad: 25 };
Object.seal(obj);

obj.edad = 30; // Esto sí es posible
delete obj.edad; // No tiene efecto
```

### Object.getOwnPropertyDescriptors()

Devuelve un objeto con todas las propiedades y sus descriptores.

```js
const descriptor = Object.getOwnPropertyDescriptors(persona);
console.log(descriptor);
// {
//   nombre: { value: 'Ana', writable: true, enumerable: true, configurable: true },
//   edad: { value: 30, writable: true, enumerable: true, configurable: true }
// }
```

### Object.hasOwnProperty()

Devuelve `true` si el objeto tiene la propiedad especificada (sin verificar en la cadena de prototipos).

```js
console.log(persona.hasOwnProperty("nombre")); // true
console.log(persona.hasOwnProperty("toString")); // false (pertenece al prototipo)
```

### Object.create()

Crea un nuevo objeto utilizando un objeto existente como el prototipo del nuevo objeto.

```js
const prototipo = { tipo: "Humano" };
const nuevoObjeto = Object.create(prototipo);
console.log(nuevoObjeto.tipo); // "Humano"
```

### Object.defineProperty()

Define una nueva propiedad en un objeto o modifica una existente, configurando su valor y características (como si es enumerable, configurable o writable).

```js
const objeto = {};
Object.defineProperty(objeto, "edad", {
  value: 25,
  writable: false,
});

objeto.edad = 30; // No tiene efecto porque "writable" es false
console.log(objeto.edad); // 25
```

### Object.is()

Comprueba si dos valores son estrictamente iguales. A diferencia de ===, también trata correctamente los casos de NaN, +0 y -0.

```js
console.log(Object.is(25, 25)); // true
console.log(Object.is(NaN, NaN)); // true (a diferencia de ===)
console.log(Object.is(+0, -0)); // false (a diferencia de ===)
```
