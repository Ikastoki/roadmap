<h1 align="center">Encapsulación</h1>

<h2>📑 Contenido</h2>

- [¿Qué es la Encapsulación?](#qué-es-la-encapsulación)
- [Implementación de la Encapsulación en JavaScript](#implementación-de-la-encapsulación-en-javascript)
  - [Convención (\_ al principio del nombre)](#convención-_-al-principio-del-nombre)
  - [Propiedades de clase privadas (#)](#propiedades-de-clase-privadas-)
- [¿Por qué es importante?](#por-qué-es-importante)
- [¿Qué son los métodos de acceso?](#qué-son-los-métodos-de-acceso)
- [Getters y Setters](#getters-y-setters)
  - [Sintaxis de un Getter](#sintaxis-de-un-getter)
  - [Sintaxis de un Setter](#sintaxis-de-un-setter)
- [¿Por qué son importantes los métodos de acceso?](#por-qué-son-importantes-los-métodos-de-acceso)

## ¿Qué es la Encapsulación?

La encapsulación es el principio de agrupar los datos y los métodos que operan sobre esos datos dentro de una única unidad, el objeto. Su propósito principal es ocultar la complejidad interna de un objeto y proteger sus datos de un acceso y modificación no deseados desde el exterior.

Imagina un coche:

- Datos encapsulados: El motor, los cilindros, los pistones, etc., son datos internos.

- Métodos públicos: El pedal del acelerador, el volante y la palanca de cambios son los métodos que usas para interactuar con el coche.

- La encapsulación asegura que no puedas acceder directamente a los pistones del motor; en su lugar, utilizas el pedal del acelerador para iniciar una serie de eventos internos que hacen que el motor funcione.

## Implementación de la Encapsulación en JavaScript

Debido a su naturaleza prototípica, JavaScript no tuvo un mecanismo de encapsulación de forma nativa durante mucho tiempo. Los desarrolladores usaban convenciones o técnicas para simularla.

### Convención (\_ al principio del nombre)

Esta es la forma más simple y común. Se utiliza un guion bajo (\_) al principio de una propiedad o método para indicar que es "privado" y que no se debería acceder a él desde fuera del objeto. Es solo una convención, el acceso sigue siendo posible.

```javascript
class Usuario {
  constructor(nombre, contrasena) {
    this._nombre = nombre; // Convención de propiedad "privada"
    this.contrasena = contrasena;
  }

  autenticar(contrasena) {
    return this.contrasena === contrasena;
  }
}

const user = new Usuario("Ana", "1234");
console.log(user._nombre); // Aún se puede acceder, pero el guion bajo avisa que no deberíamos
```

### Propiedades de clase privadas (#)

Con la llegada de ES2020, JavaScript introdujo las propiedades privadas de forma nativa usando el símbolo de almohadilla (`#`). Estas propiedades no pueden ser accedidas ni modificadas desde fuera de la clase.

```javascript
class Usuario {
  #contrasena; // Propiedad realmente privada

  constructor(nombre, contrasena) {
    this.nombre = nombre;
    this.#contrasena = contrasena; // Solo accesible dentro de la clase
  }

  autenticar(contrasena) {
    return this.#contrasena === contrasena;
  }
}

const user = new Usuario("Ana", "1234");
console.log(user.nombre); // "Ana"
// console.log(user.#contrasena); // Esto daría un error de sintaxis

console.log(user.autenticar("1234")); // true
```

En este caso, la contraseña queda protegida, y la única forma de interactuar con ella es a través del método autenticar, que es la interfaz pública del objeto.

## ¿Por qué es importante?

- Seguridad y fiabilidad: Protege los datos internos de un objeto de manipulaciones externas no deseadas, lo que reduce el riesgo de errores.

- Mantenibilidad: Hace que el código sea más fácil de mantener, ya que los cambios internos en un objeto no afectarán a la lógica externa que lo utiliza.

- Claridad: Los objetos tienen una interfaz bien definida, lo que facilita a otros desarrolladores entender cómo usarlos sin necesidad de conocer su implementación interna.

## ¿Qué son los métodos de acceso?

Los métodos de acceso son funciones que se utilizan para leer (getters) y escribir (setters) el valor de las propiedades internas de un objeto. Al usar estos métodos en lugar de acceder a las propiedades directamente, te aseguras de que se mantenga el principio de encapsulación, ya que controlas exactamente cómo se manipulan los datos.

Esto es especialmente importante porque:

- Validación: Puedes incluir lógica para validar los datos antes de asignarlos a una propiedad. Por ejemplo, un setter para una edad podría verificar que el valor sea un número positivo.

- Abstracción: Ocultas los detalles de implementación. Si decides cambiar la forma en que se almacena una propiedad internamente, solo necesitas modificar el getter y el setter, y la interfaz pública del objeto seguirá siendo la misma.

## Getters y Setters

En JavaScript, se pueden crear `getters` y `setters` con la sintaxis de clases moderna, lo que hace que el código sea más legible y se parezca a las propiedades normales.

### Sintaxis de un Getter

Un `getter` es un método que se define con la palabra clave `get`. Se usa para obtener el valor de una propiedad.

```javascript
class Producto {
  #precio; // Propiedad privada

  constructor(nombre, precio) {
    this.nombre = nombre;
    this.#precio = precio;
  }

  // Getter para obtener el precio
  get precioFinal() {
    // Puedes aplicar lógica de negocio, como un impuesto
    return this.#precio * 1.21;
  }
}

const producto = new Producto("Laptop", 1000);
console.log(producto.precioFinal); // Se accede como una propiedad, no como un método: 1210
```

En este ejemplo, la propiedad privada `#precio` solo se puede acceder a través del `getter` `precioFinal`, que además aplica una lógica de impuesto.

### Sintaxis de un Setter

Un `setter` es un método que se define con la palabra clave `set`. Se usa para asignar un valor a una propiedad.

```js
class Coche {
  #velocidad;

  constructor() {
    this.#velocidad = 0;
  }

  // Setter para establecer la velocidad
  set velocidad(nuevaVelocidad) {
    // Lógica de validación
    if (nuevaVelocidad < 0) {
      throw new Error("La velocidad no puede ser negativa.");
    }
    this.#velocidad = nuevaVelocidad;
  }
}

const miCoche = new Coche();
miCoche.velocidad = 100; // El setter se llama automáticamente

// miCoche.velocidad = -10; // Esto lanzaría un error
```

Aquí, el `setter` velocidad se encarga de validar que la velocidad asignada sea un valor válido antes de actualizar la propiedad privada #velocidad

## ¿Por qué son importantes los métodos de acceso?

- Control total: Te permiten controlar la forma en que los datos de tu objeto son leídos y escritos.

- Lógica de negocio: Puedes añadir reglas de negocio y validación en un solo lugar.

- Abstracción: Ocultan la implementación real de los datos, lo que hace que tu código sea más flexible y fácil de mantener.
