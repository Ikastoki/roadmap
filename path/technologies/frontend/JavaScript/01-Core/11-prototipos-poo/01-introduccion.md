<h1 align="center">Prototipos y POO</h1>

<h2>📑 Contenido</h2>

- [Prototipos en JavaScript](#prototipos-en-javascript)
- [Programación Orientada a Objetos (POO)](#programación-orientada-a-objetos-poo)
  - [Encapsulación](#encapsulación)
  - [Abstracción](#abstracción)
  - [Herencia](#herencia)
  - [Polimorfismo](#polimorfismo)

## Prototipos en JavaScript

A diferencia de otros lenguajes de programación basados en clases (como Java o C++), JavaScript utiliza un modelo de herencia basado en prototipos. Esto significa que los objetos no heredan de "clases", sino que heredan directamente de otros objetos.

- ¿Qué es un prototipo?: Cada objeto en JavaScript tiene una propiedad interna llamada `[[Prototype]]` (o `__proto__` en implementaciones más antiguas) que apunta a otro objeto.

- La Cadena de Prototipos: Cuando intentas acceder a una propiedad o a un método en un objeto, JavaScript primero busca esa propiedad en el objeto mismo. Si no la encuentra, sigue la cadena de prototipos y la busca en el objeto al que apunta su `__proto__`. Este proceso continúa hasta que encuentra la propiedad o llega al final de la cadena (donde el prototipo es `null`).

- Constructor: Las "clases" en JavaScript son en realidad funciones constructoras. La propiedad `prototype` de una función constructora es el objeto que se usará como prototipo para todas las instancias creadas con `new`.

## Programación Orientada a Objetos (POO)

Aunque JavaScript no se basa en clases de forma nativa, se pueden aplicar los principios de la POO para organizar y estructurar el código de manera efectiva. La POO se basa en cuatro principios fundamentales:

### Encapsulación

La encapsulación se trata de ocultar la complejidad y los datos de un objeto para que no sean accesibles o modificables desde fuera. Esto reduce errores y hace que el código sea más seguro y modular. En JavaScript, se puede simular la encapsulación con clausuras (`closures`), símbolos o, más recientemente, con propiedades de clase privadas (`#`).

Ejemplo: En una clase, los métodos públicos son la única forma de interactuar con las propiedades internas (`\_nombre` o `#nombre`), que no son accesibles directamente.

### Abstracción

La abstracción se refiere a exponer solo la funcionalidad necesaria para el usuario del objeto, ocultando los detalles internos de cómo funciona. Piensa en un control remoto de televisión: solo necesitas saber qué hace el botón de "encender", no cómo funciona el circuito interno para lograrlo.

Ejemplo: Un método `guardarUsuario()` en una clase, donde solo necesitas llamarlo sin preocuparte por la lógica interna que maneja la base de datos o la validación.

### Herencia

La herencia permite crear nuevos objetos (clases) basados en objetos existentes, heredando sus propiedades y métodos. Esto fomenta la reutilización del código. En JavaScript, la herencia se logra a través de la cadena de prototipos. Con la sintaxis de `class`, se usa la palabra clave `extends`.

Ejemplo: Una clase `Coche` puede heredar de una clase `Vehiculo`, adquiriendo métodos como `encenderMotor()` y propiedades como ruedas.

### Polimorfismo

El polimorfismo significa que un mismo método puede tener diferentes comportamientos en diferentes objetos. La palabra significa "muchas formas". En la POO, permite que los objetos de diferentes clases respondan al mismo mensaje (método) de una manera propia.

Ejemplo: Una clase `Gato` y una clase `Perro` pueden tener un método `hacerSonido()`. Al llamar a este método, el `Gato` dirá "Miau" y el `Perro` dirá "Guau". A pesar de que el nombre del método es el mismo, el comportamiento es distinto.
