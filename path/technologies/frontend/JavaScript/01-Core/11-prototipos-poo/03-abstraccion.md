<h1 align="center">Abstracción</h1>

<h2>📑 Contenido</h2>

- [¿Qué es la Abstracción?](#qué-es-la-abstracción)
- [Implementación de la Abstracción en JavaScript](#implementación-de-la-abstracción-en-javascript)
- [Abstracción vs. Encapsulación](#abstracción-vs-encapsulación)

## ¿Qué es la Abstracción?

La abstracción es el principio de mostrar solo la información relevante al usuario y ocultar los detalles internos. Se trata de crear una interfaz simple y clara para interactuar con un objeto, sin necesidad de que el usuario conozca su funcionamiento interno. La abstracción es, en esencia, la simplificación.

Un ejemplo perfecto es el de un teléfono móvil:

- Abstracción: No necesitas saber cómo se procesan las señales, se gestiona la batería o se almacenan los datos internamente. Solo necesitas conocer la interfaz de la pantalla táctil y los botones para hacer una llamada o usar una aplicación.

- Los detalles complejos están abstraídos detrás de una interfaz de usuario simple.

## Implementación de la Abstracción en JavaScript

En JavaScript, la abstracción se logra de varias maneras:

- Encapsulación: La encapsulación (con propiedades privadas `#`) es la base de la abstracción. Al ocultar los datos y exponer solo los métodos públicos (getters y setters), ya estás practicando la abstracción. El usuario solo interactúa con la interfaz pública del objeto.

- Interfaces (y clases abstractas): Aunque JavaScript no tiene interfaces nativas como otros lenguajes, la idea de una interfaz es un concepto de abstracción clave. Una interfaz define un contrato, una serie de métodos que una clase debe implementar, sin especificar cómo lo hará. Se pueden simular con clases abstractas, que no pueden ser instanciadas por sí mismas y deben ser extendidas.

```javascript
class ReproductorDeMusica {
  #canciones;

  constructor() {
    this.#canciones = [];
  }

  // Método público que representa la interfaz
  agregarCancion(cancion) {
    this.#canciones.push(cancion);
    console.log(`${cancion} ha sido agregada.`);
  }

  // Método público que es la interfaz para el usuario
  reproducir() {
    // La lógica interna de cómo se reproduce la música está oculta
    if (this.#canciones.length > 0) {
      console.log(`Reproduciendo: ${this.#canciones[0]}`);
    } else {
      console.log("No hay canciones en la lista.");
    }
  }
}

const miReproductor = new ReproductorDeMusica();
miReproductor.agregarCancion("Bohemian Rhapsody"); // La interfaz es simple
miReproductor.reproducir(); // Solo se invoca el método, sin saber su funcionamiento interno
```

En este ejemplo, la propiedad privada `#canciones` está encapsulada. La abstracción se ve en el método `reproducir()`, donde el usuario no necesita saber cómo se gestiona el array de canciones ni cómo se emite el sonido. Solo necesita invocar el método para que la música empiece a sonar.

## Abstracción vs. Encapsulación

Es común confundir la abstracción y la encapsulación, ya que están estrechamente relacionadas:

- Encapsulación: Es la técnica de agrupar datos y métodos y restringir el acceso a los datos. Es un mecanismo de protección.

- Abstracción: Es el concepto de simplificar y exponer solo una interfaz relevante. Es un principio de diseño.

> [!NOTE]
> La encapsulación es una forma de lograr la abstracción.
