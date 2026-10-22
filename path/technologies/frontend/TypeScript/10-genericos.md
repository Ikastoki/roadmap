<h1 align="center">Genéricos</h1>

<h2>📑 Contenido</h2>

- [Genéricos](#genéricos)

## Genéricos

Los genéricos (o "Generics") son una característica que permite crear componentes que pueden trabajar sobre un conjunto de tipos en lugar de uno solo. Esto proporciona una mayor flexibilidad y reutilización de código al escribir funciones, clases y otros tipos de estructuras.

## Generic Types (Tipos Genéricos)

Los tipos genéricos son aquellos que están parametrizados por uno o varios tipos. Se definen utilizando un tipo de parámetro entre ángulos (<>). Esto permite que una función o una clase puedan trabajar con diferentes tipos de datos de manera dinámica.

```ts
function identidad<T>(arg: T): T {
  return arg;
}

let numero: number = identidad<number>(5); // Tipo explícito
let texto: string = identidad("Hola"); // Tipo inferido
```

## Generic Constraints (Restricciones Genéricas)

Las restricciones genéricas son utilizadas para restringir los tipos que pueden ser utilizados como argumentos genéricos. Esto se hace mediante la aplicación de restricciones a los tipos de parámetros genéricos, lo que garantiza que ciertos métodos y propiedades estén disponibles en los tipos especificados.

```ts
interface Longitud {
  length: number;
}

function contarCaracteres<T extends Longitud>(valor: T): number {
  return valor.length;
}

let longitudTexto: number = contarCaracteres("Hola");
let longitudArray: number = contarCaracteres([1, 2, 3, 4]);
```
