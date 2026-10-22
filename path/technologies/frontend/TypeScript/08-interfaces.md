<h1 align="center">Interfaces</h1>

<h2>📑 Contenido</h2>

- [Interfaces](#interfaces)
- [Types vs Interfaces](#types-vs-interfaces)
- [Extending Interfaces](#extending-interfaces)
- [Interfaces Declaration](#interfaces-declaration)
- [Hybrid Types](#hybrid-types)

## Interfaces

Las interfaces son una forma poderosa de definir la estructura de un objeto, especificando qué propiedades y métodos debe tener.

Permiten definir la forma de un objeto. Se utilizan para describir la estructura de los objetos y garantizar que los objetos cumplan con ciertas reglas de tipado.

```ts
interface Persona {
  nombre: string;
  edad: number;
  saludar(): void;
}

let persona: Persona = {
  nombre: "Juan",
  edad: 30,
  saludar() {
    console.log(`Hola, mi nombre es ${this.nombre}.`);
  },
};

persona.saludar(); // Salida: Hola, mi nombre es Juan.
```

## Types vs Interfaces

Tanto type como interface pueden usarse para definir formas de objetos. Sin embargo, hay diferencias sutiles entre ellos. Mientras que las interfaces se utilizan principalmente para definir la forma de los objetos, los tipos (type) pueden hacer muchas otras cosas, como definir tipos de unión, tipos de intersección, alias de tipos, etc.

```ts
type Empleado = {
  nombre: string;
  salario: number;
};

interface Gerente extends Empleado {
  departamento: string;
}
```

## Extending Interfaces

Las interfaces pueden extender otras interfaces para heredar propiedades y métodos. Esto permite crear jerarquías de interfaces que facilitan la reutilización de código.

```ts
interface Animal {
  nombre: string;
}

interface Mascota extends Animal {
  tipo: string;
}
```

## Interfaces Declaration

Las interfaces también se pueden declarar de otras maneras, como tipo de declaración. Esto permite utilizar la misma interfaz en múltiples archivos sin tener que importarla explícitamente.

```ts
// archivo1.ts
declare interface Coche {
  marca: string;
  modelo: string;
}

// archivo2.ts
let miCoche: Coche = { marca: "Toyota", modelo: "Corolla" };
```

## Hybrid Types

Las interfaces híbridas son aquellas que combinan propiedades y métodos en una sola interfaz. Esto permite modelar objetos que tienen comportamientos y características.

```ts
interface Computadora {
  encendida: boolean;
  iniciar(): void;
  apagar(): void;
}
```
