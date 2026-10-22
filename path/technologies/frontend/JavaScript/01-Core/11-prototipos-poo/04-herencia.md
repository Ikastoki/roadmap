<h1 align="center">Herencia</h1>

<h2>📑 Contenido</h2>

- [¿Qué es la Herencia?](#qué-es-la-herencia)
- [Herencia en JavaScript](#herencia-en-javascript)
- [Usando extends y super](#usando-extends-y-super)
- [Herencia Prototípica vs. Herencia Clásica](#herencia-prototípica-vs-herencia-clásica)
- [Composición vs. Herencia](#composición-vs-herencia)
  - [La Herencia](#la-herencia)
- [La Composición](#la-composición)
  - [Composición en JavaScript (Ejemplo)](#composición-en-javascript-ejemplo)

## ¿Qué es la Herencia?

La herencia es un mecanismo que permite a un objeto (**clase hija**) adquirir las propiedades y los métodos de otro objeto (**clase padre**). Esto promueve la reutilización del código, ya que no tienes que reescribir la misma lógica en múltiples lugares.

Imagina una jerarquía:

Un `Animal` tiene propiedades y métodos comunes como `nombre` y `comer()`.

Un `Perro` y un `Gato` son tipos específicos de `Animales`.

A través de la herencia, `Perro` y `Gato` heredan las propiedades y métodos de `Animal`, y pueden añadir sus propias características únicas (como `ladrar() o maullar()`).

## Herencia en JavaScript

En JavaScript, la herencia se implementa a través de la cadena de prototipos, pero la sintaxis moderna de clases (`class`) hace que sea mucho más fácil de entender y utilizar.

## Usando extends y super

La forma más común de implementar la herencia hoy en día es con las palabras clave `extends` y `super`.

- `extends`: Se usa en la declaración de la clase hija para indicar de qué clase padre va a heredar.

- `super`: Se usa dentro del constructor y los métodos de la clase hija para llamar al constructor o los métodos de la clase padre.

```javascript
// Clase Padre
class Animal {
  constructor(nombre) {
    this.nombre = nombre;
  }

  comer() {
    console.log(`${this.nombre} está comiendo.`);
  }
}

// Clase Hija que hereda de Animal
class Perro extends Animal {
  constructor(nombre, raza) {
    super(nombre); // Llama al constructor de la clase padre
    this.raza = raza;
  }

  ladrar() {
    console.log(`${this.nombre} está ladrando.`);
  }

  // Sobrescribimos el método 'comer' del padre
  comer() {
    super.comer(); // Llamamos primero al método comer del padre
    console.log("...con mucho apetito.");
  }
}

const miPerro = new Perro("Max", "Labrador");
miPerro.comer(); // Salida: Max está comiendo. ...con mucho apetito.
miPerro.ladrar(); // Salida: Max está ladrando.
```

En este ejemplo, la clase `Perro` hereda la propiedad `nombre` y el método `comer()` de la clase `Animal`. En el constructor, `super(nombre)` se encarga de inicializar la parte del objeto que corresponde a la clase padre.

## Herencia Prototípica vs. Herencia Clásica

Es importante recordar que la herencia en JavaScript es, en esencia, prototípica. La sintaxis de clases que vimos (`class`, `extends`, `super`) es solo una "azúcar sintáctica" que nos permite escribir un código que se parece a la herencia clásica, pero el motor de JavaScript sigue funcionando con la cadena de prototipos.

## Composición vs. Herencia

### La Herencia

La herencia crea una relación de tipo "es un" (is a). Por ejemplo, un Perro es un Animal. La clase Perro hereda las propiedades y métodos de la clase Animal, y luego añade sus propias características.

**Ventajas:**

- Simpleza: Es muy intuitiva para crear jerarquías lógicas.

- Reutilización: Se pueden reutilizar fácilmente los métodos de la clase padre.

**Desventajas:**

- Acoplamiento rígido: La clase hija está fuertemente acoplada a la clase padre. Si cambias algo en la clase padre, puede romper el funcionamiento de las clases hijas.

- Rigidez: No se puede heredar de múltiples clases en JavaScript (herencia simple). Si quieres que un objeto tenga funcionalidades de dos jerarquías diferentes, la herencia se vuelve complicada. Esto se conoce como el "problema del diamante" en otros lenguajes.

## La Composición

La composición crea una relación de tipo "tiene un" (has a). En lugar de heredar de otra clase, un objeto se construye combinando otros objetos más pequeños. Cada objeto interno se encarga de una funcionalidad específica. Por ejemplo, en lugar de que un Coche herede un Motor, el Coche tiene un objeto Motor.

**Ventajas:**

- Flexibilidad: Es muy flexible. Puedes combinar y reutilizar diferentes funcionalidades como si fueran "bloques de construcción", creando objetos con comportamientos muy específicos.

- Bajo acoplamiento: La clase principal no está ligada a la implementación de los objetos que la componen. Si un componente cambia, los demás no se ven afectados.

**Desventajas:**

- Sintaxis más compleja: Puede requerir un poco más de código al principio para conectar los diferentes componentes.

### Composición en JavaScript (Ejemplo)

Imagina un Personaje que tiene la capacidad de volar y nadar. Con la herencia, esto sería complicado. Con la composición, podemos crear objetos que representan esas habilidades y luego "componerlos" en el Personaje.

```javascript
// Habilidad de Volar (componente)
const volador = (estado) => ({
  volar: () => {
    console.log(`${estado.nombre} está volando.`);
  },
});

// Habilidad de Nadar (componente)
const nadador = (estado) => ({
  nadar: () => {
    console.log(`${estado.nombre} está nadando.`);
  },
});

// Componemos el objeto Personaje
const crearPersonaje = (nombre) => {
  const estado = { nombre };
  return {
    ...estado,
    ...volador(estado), // Le damos la habilidad de volar
    ...nadador(estado), // Y también la de nadar
  };
};

const superheroe = crearPersonaje("Superman");
superheroe.volar(); // Salida: Superman está volando.
superheroe.nadar(); // Salida: Superman está nadando.
```

Este es un patrón de diseño muy común en JavaScript llamado "mixins" o "compositor".
