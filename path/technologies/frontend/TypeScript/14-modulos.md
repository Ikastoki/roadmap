<h1 align="center">Módulos</h1>

<h2>📑 Contenido</h2>

- [Módulos](#módulos)

## Módulos

Los módulos son una forma de organizar y estructurar el código en unidades más pequeñas y separadas, lo que facilita la reutilización y el mantenimiento del código.

Archivos que contienen código que se puede importar y exportar para su uso en otros archivos. Permiten dividir el código en unidades lógicas y proporcionan un mecanismo para encapsular la funcionalidad y evitar conflictos de nombres.

## Namespaces

Los espacios de nombres (namespaces) son una forma de agrupar lógicamente un conjunto de tipos, funciones y otras entidades bajo un nombre común. Permiten evitar colisiones de nombres y organizar el código de manera jerárquica.

```ts
// En namespaceA.ts
namespace Espacio {
  export function saludar(nombre: string) {
    console.log(`Hola, ${nombre}!`);
  }
}

// En namespaceB.ts
///<reference path="namespaceA.ts" />
Espacio.saludar("Pedro"); // Salida: Hola, Pedro!
```

## Namespaces Augmentation

La extensión de namespaces (namespaces augmentation) es una técnica que permite agregar nuevas declaraciones a un espacio de nombres existente desde diferentes archivos y ubicaciones sin modificar directamente el archivo original. Esto es útil para extender la funcionalidad de bibliotecas externas.

```ts
// En namespaceA.ts
namespace Espacio {
  export function saludar(nombre: string) {
    console.log(`Hola, ${nombre}!`);
  }
}

// En namespaceAExtension.ts
declare namespace Espacio {
  function despedirse(nombre: string): void;
}

Espacio.despedirse = function (nombre: string) {
  console.log(`Adiós, ${nombre}!`);
};

// En main.ts
///<reference path="namespaceA.ts" />
///<reference path="namespaceAExtension.ts" />
Espacio.saludar("Ana"); // Salida: Hola, Ana!
Espacio.despedirse("Lucas"); // Salida: Adiós, Lucas!
```

## Global Augmentation

La extensión global (global augmentation) es similar a la extensión de namespaces, pero en lugar de aplicarse a un namespace, se aplica a objetos globales como Array, String, Number, etc. Permite agregar nuevas propiedades o métodos a estos objetos globales.

```ts
// En globalExtensions.ts
interface Array<T> {
  primero(): T;
}

Array.prototype.primero = function () {
  return this[0];
};

// En main.ts
///<reference path="globalExtensions.ts" />
let numeros: number[] = [1, 2, 3];
console.log(numeros.primero()); // Salida: 1
```

## Ambient Module

Un módulo ambiental (ambient module) es una declaración que describe un módulo existente en otro lugar, como una biblioteca JavaScript externa. Se utiliza para proporcionar tipos y declaraciones para bibliotecas que no están escritas en TypeScript.

```ts
// En lodash.d.ts (archivo de declaración de módulo ambiental)
declare module "lodash" {
  export function chunk<T>(array: T[], size: number): T[][];
}

// En main.ts
import * as _ from "lodash";

let numeros: number[] = [1, 2, 3, 4, 5];
let chunks = _.chunk(numeros, 2);
console.log(chunks); // Salida: [[1, 2], [3, 4], [5]]
```

## External Module

Un módulo externo (external module) es un archivo de TypeScript que define un módulo que puede ser importado y utilizado por otros archivos de TypeScript o JavaScript. Estos módulos pueden contener exportaciones de código y dependencias externas.

```ts
// En moduloA.ts
export function saludar(nombre: string) {
  console.log(`Hola, ${nombre}!`);
}

// En moduloB.ts
import { saludar } from "./moduloA";

saludar("Juan"); // Salida: Hola, Juan!
```
