<h1 align="center">Tipos de Objetos</h1>

<h2>📑 Contenido</h2>

- [Tipos de Objetos](#tipos-de-objetos)
- [Interface (Interfaz)](#interface-interfaz)
- [Class (Clase)](#class-clase)
- [Enum (Enumeración)](#enum-enumeración)
- [Array (Arreglo)](#array-arreglo)
- [Tuple (Tupla)](#tuple-tupla)
- [object](#object)
- [Any](#any)
- [Never](#never)
- [Unknown](#unknown)

## Tipos de Objetos

Los "Object Types" (Tipos de Objeto) en TypeScript se refieren a las estructuras de datos que representan objetos en el código

## Interface (Interfaz)

Una interfaz en TypeScript define la forma que debe tener un objeto. Es una estructura puramente declarativa que especifica un conjunto de propiedades y métodos que un objeto debe implementar. Las interfaces son útiles para definir contratos en el código y para establecer una estructura común entre diferentes partes del programa.

```ts
interface Persona {
  nombre: string;
  edad: number;
  saludar(): void;
}

// Implementación de la interfaz
const persona: Persona = {
  nombre: "Juan",
  edad: 30,
  saludar() {
    console.log("¡Hola!");
  },
};

persona.saludar(); // Salida: ¡Hola!
```

## Class (Clase)

Una clase en TypeScript es una plantilla para crear objetos que define atributos y métodos comunes a esos objetos. Permite crear instancias de objetos basadas en esa plantilla.

```ts
class Animal {
  nombre: string;
  constructor(nombre: string) {
    this.nombre = nombre;
  }
  hacerSonido() {
    console.log("Haciendo sonido...");
  }
}

// Crear una instancia de la clase Animal
const perro = new Animal("Bobby");
perro.hacerSonido(); // Salida: Haciendo sonido...
```

## Enum (Enumeración)

Un enum en TypeScript es una forma de definir un conjunto de constantes con nombres más descriptivos. Cada miembro de un enum tiene un valor asociado que puede ser numérico o de cadena.

```ts
enum Direccion {
  Arriba,
  Abajo,
  Izquierda,
  Derecha,
}

console.log(Direccion.Arriba); // Salida: 0
console.log(Direccion[Direccion.Arriba]); // Salida: Arriba
```

## Array (Arreglo)

Un arreglo en TypeScript es una colección ordenada de elementos del mismo tipo. Los arreglos en TypeScript se pueden declarar usando sintaxis de corchetes o usando la clase Array.

```ts
const numeros: number[] = [1, 2, 3, 4, 5];
console.log(numeros); // Salida: [1, 2, 3, 4, 5]
```

## Tuple (Tupla)

Una tupla en TypeScript es una matriz de tipos fijos donde cada elemento de la matriz puede ser de un tipo diferente. Las tuplas permiten representar una colección de valores heterogéneos.

```ts
let tupla: [string, number];
tupla = ["Hola", 10]; // Valores válidos
// tupla = [10, "Hola"]; // Error: Los tipos no coinciden

console.log(tupla[0]); // Salida: Hola
console.log(tupla[1]); // Salida: 10
```

## object

El tipo object, que representa cualquier objeto no primitivo. A diferencia del tipo Object, que se refiere al tipo de dato de los objetos en JavaScript, el tipo object en TypeScript se refiere a cualquier valor no primitivo. Esto incluye objetos regulares, arrays, funciones, clases, y cualquier otro tipo de objeto **que no sea un tipo primitivo como number, string, boolean, null o undefined.**

```ts
let objeto: object;

objeto = {}; // Objeto regular
objeto = []; // Arreglo
objeto = new Date(); // Instancia de una clase
objeto = () => {}; // Función
```

## Any

El tipo any en TypeScript representa cualquier tipo de valor. Se utiliza cuando no se conoce el tipo de una variable en tiempo de compilación o cuando se quiere optar por la comprobación de tipos.

```ts
let variable: any = 10;
variable = "Hola";
variable = true;
```

## Never

El tipo never representa un tipo de valor que nunca ocurre. Se utiliza para funciones que nunca terminan de ejecutarse o para funciones que siempre lanzan una excepción.

```ts
function error(mensaje: string): never {
  throw new Error(mensaje);
}

function bucleInfinito(): never {
  while (true) {}
}
```

## Unknown

Similar a any, unknown es un tipo de valor que representa cualquier tipo, pero con restricciones más estrictas. No se puede realizar ninguna operación en un valor de tipo unknown sin primero realizar una comprobación de tipo o una conversión de tipo.

```ts
let valor: unknown;
valor = 10; // Válido
valor = "Hola"; // Válido

// No se pueden realizar operaciones sin comprobación de tipo
// let longitud = valor.length; // Error: length no es válido en tipo 'unknown'

// Comprobación de tipo
if (typeof valor === "string") {
  let longitud = valor.length; // Válido dentro del bloque if
}
```
