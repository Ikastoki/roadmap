<h1 align="center">Polimorfismo</h1>

<h2>📑 Contenido</h2>

- [¿Qué es el Polimorfismo?](#qué-es-el-polimorfismo)
- [Tipos de Polimorfismo](#tipos-de-polimorfismo)
  - [Sobreescritura de Métodos (Method Overriding)](#sobreescritura-de-métodos-method-overriding)
  - [Sobrecarga de Métodos (Method Overloading)](#sobrecarga-de-métodos-method-overloading)

## ¿Qué es el Polimorfismo?

El polimorfismo es la capacidad de un objeto para tomar diferentes formas, o más específicamente, la capacidad de los objetos de diferentes clases para responder al mismo mensaje (método) de una manera propia y única.

Imagina que tienes una clase Figura con un método dibujar(). Luego, tienes clases hijas como Circulo y Cuadrado que heredan de Figura. Cada una de ellas implementará el método dibujar() de manera diferente: el Círculo dibujará una forma redonda, mientras que el Cuadrado dibujará una forma de cuatro lados.

Cuando se invoca el método dibujar() en cualquiera de las instancias, el programa sabrá automáticamente qué versión del método ejecutar, incluso si la variable se declara como un tipo Figura.

## Tipos de Polimorfismo

En el contexto de JavaScript y la POO, el polimorfismo se manifiesta de dos formas principales:

- Sobreescritura: Se da en la herencia, cuando un método de la clase hija redefine un método de la clase padre. JavaScript lo soporta de forma nativa.

- Sobrecarga: Múltiples métodos con el mismo nombre, pero diferente firma (parámetros). JavaScript no lo soporta de forma nativa, pero se puede simular.

### Sobreescritura de Métodos (Method Overriding)

Este es el tipo más común de polimorfismo en JavaScript. Ocurre cuando una clase hija define su propia implementación de un método que ya existe en la clase padre. La clase hija "sobrescribe" el método de la clase padre.

```javascript
class Animal {
  hacerSonido() {
    console.log("El animal hace un sonido.");
  }
}

class Gato extends Animal {
  // Sobreescritura del método hacerSonido()
  hacerSonido() {
    console.log("El gato hace miau.");
  }
}

class Perro extends Animal {
  // Sobreescritura del método hacerSonido()
  hacerSonido() {
    console.log("El perro hace guau.");
  }
}

const miGato = new Gato();
const miPerro = new Perro();

miGato.hacerSonido(); // Salida: El gato hace miau.
miPerro.hacerSonido(); // Salida: El perro hace guau.
```

Ambos objetos, miGato y miPerro, responden al mismo mensaje (hacerSonido()) de forma diferente, demostrando el polimorfismo.

### Sobrecarga de Métodos (Method Overloading)

La sobrecarga de métodos ocurre cuando tienes múltiples métodos con el mismo nombre, pero con diferentes parámetros (número de argumentos o tipo de datos). JavaScript no tiene sobrecarga de métodos nativa como C++ o Java. Si defines dos funciones con el mismo nombre, la última que se declare sobrescribirá a la anterior.

Sin embargo, se puede simular la sobrecarga en JavaScript con condicionales o argumentos por defecto.

```javascript
function mostrarInfo(param1, param2) {
  if (param2 === undefined) {
    console.log(`Hola, ${param1}`); // Lógica para un solo argumento
  } else {
    console.log(`Hola, ${param1} y ${param2}`); // Lógica para dos argumentos
  }
}

mostrarInfo("Ana"); // Salida: Hola, Ana
mostrarInfo("Ana", "Luis"); // Salida: Hola, Ana y Luis
```
