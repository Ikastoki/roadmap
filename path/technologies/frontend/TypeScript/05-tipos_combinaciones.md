<h1 align="center">Combinación de Tipos</h1>

<h2>📑 Contenido</h2>

- [Combinación de Tipos](#combinación-de-tipos)

## Combinación de Tipos

Los "Combining Types" (Combinación de Tipos) en TypeScript se refieren a la capacidad de crear tipos compuestos combinando múltiples tipos existentes.

## Union Types (Tipos de Unión)

Los union types permiten combinar varios tipos en uno solo. Se utilizan para representar un valor que puede ser de uno de varios tipos diferentes. Los tipos de unión se crean usando el operador de unión |.

```ts
let resultado: number | string;
resultado = 10; // Válido
resultado = "Diez"; // Válido
// resultado = true; // Inválido, solo number o string son permitidos
```

## Intersection Types (Tipos de Intersección)

Los intersection types permiten combinar varios tipos en uno solo, donde el nuevo tipo contiene todas las propiedades de los tipos originales. Se utilizan para representar un valor que cumple con todas las características de varios tipos.

```ts
interface A {
  prop1: number;
}

interface B {
  prop2: string;
}

type C = A & B;

let objeto: C = {
  prop1: 10,
  prop2: "Hola",
};
```

## Type Aliases (Alias de Tipos)

Los type aliases permiten darle un nombre a un tipo existente o a una combinación de tipos para poder reutilizarlo en múltiples partes del código.

```ts
type NumeroOTexto = number | string;

let valor: NumeroOTexto;
valor = 10; // Válido
valor = "Diez"; // Válido
```

## Keyof Operator (Operador Keyof)

El operador keyof se utiliza para obtener las claves (o nombres de las propiedades) de un tipo. Se puede utilizar para crear tipos más genéricos o para realizar operaciones con las claves de un objeto.

```ts
interface Persona {
  nombre: string;
  edad: number;
}

type ClavesDePersona = keyof Persona;

let clave: ClavesDePersona = "nombre"; // Válido
// let clave: ClavesDePersona = "direccion"; // Inválido, "direccion" no es una clave de Persona
```
