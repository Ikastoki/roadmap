<h1 align="center">Clases</h1>

<h2>📑 Contenido</h2>

- [Clases](#clases)
- [Constructor Params](#constructor-params)
- [Constructor Overloading](#constructor-overloading)
- [Acces Modifiers](#acces-modifiers)
- [Abstract Classes](#abstract-classes)
- [Inheritance vs Polymorphism](#inheritance-vs-polymorphism)
- [Method Override](#method-override)

## Clases

Las clases son una estructura fundamental para la programación orientada a objetos. Permiten definir plantillas para crear objetos que tienen propiedades y métodos comunes.

```ts
class Persona {
  nombre: string;

  constructor(nombre: string) {
    this.nombre = nombre;
  }

  saludar(): void {
    console.log(`Hola, mi nombre es ${this.nombre}.`);
  }
}

let persona = new Persona("Juan");
persona.saludar(); // Salida: Hola, mi nombre es Juan.
```

## Constructor Params

Los parámetros del constructor son utilizados para inicializar las propiedades de un objeto al momento de crearlo.

```ts
class Rectangulo {
  ancho: number;
  altura: number;

  constructor(ancho: number, altura: number) {
    this.ancho = ancho;
    this.altura = altura;
  }
}

let rectangulo = new Rectangulo(10, 5);
```

## Constructor Overloading

TypeScript no permite la sobrecarga de métodos o constructores de forma directa como en otros lenguajes, pero puedes simularlo utilizando valores opcionales o valores por defecto en los parámetros.

```ts
class Circulo {
  constructor(public radio: number, public color: string = "rojo") {
    // Constructor con un parámetro por defecto
  }
}

let circulo1 = new Circulo(5);
let circulo2 = new Circulo(3, "azul");
```

## Acces Modifiers

Los modificadores de acceso (public, private, protected) permiten controlar la visibilidad de las propiedades y métodos de una clase.

```ts
class Auto {
  public marca: string;
  private modelo: string;
  protected año: number;

  constructor(marca: string, modelo: string, año: number) {
    this.marca = marca;
    this.modelo = modelo;
    this.año = año;
  }
}

let miAuto = new Auto("Toyota", "Corolla", 2022);
console.log(miAuto.marca); // Acceso permitido
// console.log(miAuto.modelo); // Error: modelo es privado
// console.log(miAuto.año); // Error: año es protegido
```

## Abstract Classes

Las clases abstractas sirven como plantillas para otras clases. No pueden ser instanciadas directamente, pero pueden ser extendidas por otras clases.

```ts
abstract class Figura {
  abstract calcularArea(): number;
}

class Rectangulo extends Figura {
  constructor(private ancho: number, private altura: number) {
    super();
  }

  calcularArea(): number {
    return this.ancho * this.altura;
  }
}
```

## Inheritance vs Polymorphism

La herencia permite que una clase herede propiedades y métodos de otra clase. El polimorfismo permite que objetos de diferentes clases se traten de la misma manera.

```ts
class Animal {
  sonido(): void {
    console.log("Haciendo sonido...");
  }
}

class Perro extends Animal {
  sonido(): void {
    console.log("Guau!");
  }
}

let miMascota: Animal = new Perro();
miMascota.sonido(); // Salida: Guau!
```

## Method Override

El "method override" es la capacidad de una subclase para proporcionar una implementación específica de un método que ya está definido en una superclase.

```ts
class Vehiculo {
  mover(): void {
    console.log("Moviéndose...");
  }
}

class Auto extends Vehiculo {
  mover(): void {
    console.log("Conduciendo el auto...");
  }
}

let miAuto: Vehiculo = new Auto();
miAuto.mover(); // Salida: Conduciendo el auto...
```
