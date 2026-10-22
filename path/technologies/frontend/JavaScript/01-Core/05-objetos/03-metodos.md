<h1 align='center'>Métodos</h1>

<h2>📑 Contenido</h2>

- [Métodos](#métodos)
- [Métodos con Parámetros](#métodos-con-parámetros)
- [Uso de this en Métodos:](#uso-de-this-en-métodos)

## Métodos

En JavaScript, un método de un objeto es simplemente una función que se almacena como una propiedad dentro de un objeto. Estos métodos permiten que los objetos tengan comportamientos o acciones asociados con ellos. Los métodos pueden ser cualquier función, incluidas las funciones anónimas y las funciones flecha.

```js
let coche = {
  marca: "Toyota",
  modelo: "Corolla",
  arrancar: function () {
    console.log("El coche ha arrancado");
  },
};

coche.arrancar(); // Llama al método y ejecuta el código
```

En este ejemplo, el objeto coche tiene una propiedad arrancar que es una función, y al llamar a coche.arrancar(), se ejecuta el método que imprime un mensaje en la consola.

## Métodos con Parámetros

Los métodos también pueden aceptar parámetros, igual que las funciones normales.

```js
let calculadora = {
  sumar: function (a, b) {
    return a + b;
  },
};

console.log(calculadora.sumar(3, 4)); // Imprime 7
```

## Uso de this en Métodos:

Dentro de un método, puedes usar la palabra clave `this` para referirte al propio objeto. Esto es útil cuando un método necesita acceder a las propiedades del objeto en el que está definido.

```js
let persona = {
  nombre: "Ana",
  edad: 25,
  saludar: function () {
    console.log("Hola, mi nombre es " + this.nombre);
  },
};

persona.saludar(); // Imprime "Hola, mi nombre es Ana"
```

Aquí, `this.nombre` hace referencia a la propiedad `nombre` del objeto `persona`.
