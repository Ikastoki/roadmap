<h1 align="center">Decoradores</h1>

<h2>📑 Contenido</h2>

- [Decoradores](#decoradores)

## Decoradores

Los decoradores son una característica de TypeScript que proporciona una forma de añadir metadatos y modificar comportamientos en clases, métodos, propiedades y otros elementos del lenguaje de manera declarativa. Los decoradores se utilizan comúnmente en aplicaciones TypeScript y en el ecosistema de Angular para realizar tareas como la inyección de dependencias, la validación de datos, el enrutamiento, la administración de estados y más.

Los decoradores se definen utilizando la sintaxis @nombreDelDecorador justo antes de la declaración del elemento que se va a decorar. Un decorador es simplemente una función que acepta uno o más argumentos y devuelve una función, la cual es llamada con el objeto de destino y otras informaciones relevantes en tiempo de ejecución.

```ts
// Decorador de método
function miDecorador(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  console.log(
    `Decorando el método ${propertyKey} de la clase ${target.constructor.name}`
  );
}

class MiClase {
  @miDecorador
  miMetodo() {
    console.log("Ejecutando miMetodo");
  }
}

let instancia = new MiClase();
instancia.miMetodo();
```

En este ejemplo, @miDecorador es un decorador de método que se aplica al método miMetodo de la clase MiClase. Cuando se instancia MiClase y se llama a miMetodo(), el decorador se ejecuta y muestra un mensaje en la consola.
