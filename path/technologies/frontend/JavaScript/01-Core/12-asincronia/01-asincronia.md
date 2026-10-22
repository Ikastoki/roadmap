<h1 align='center'>Asincronía</h1>

<h2>📑 Contenido</h2>

- [Asincronía](#asincronía)
- [Operaciones Sincrónicas vs Asincrónicas](#operaciones-sincrónicas-vs-asincrónicas)
  - [Sincronía](#sincronía)
  - [Asincronía](#asincronía-1)
- [Mecanismos de Control Asincrónico en JavaScript](#mecanismos-de-control-asincrónico-en-javascript)
  - [Callbacks](#callbacks)
  - [Promesas](#promesas)
  - [async / await](#async--await)

## Asincronía

La asincronía es un concepto fundamental en JavaScript que permite que el código realice operaciones sin bloquear la ejecución de otras tareas. Esto es esencial cuando se trabaja con operaciones que toman tiempo en completarse, como solicitudes de red (AJAX), temporizadores, o la manipulación de archivos. JavaScript está diseñado para ser un lenguaje monohilo, lo que significa que solo puede ejecutar una tarea a la vez en su hilo principal. Sin embargo, utiliza modelos asincrónicos para mejorar la eficiencia, evitando que tareas largas bloqueen el flujo del programa.

## Operaciones Sincrónicas vs Asincrónicas

### Sincronía

Las operaciones sincrónicas se ejecutan de manera secuencial, es decir, cada operación debe completarse antes de que la siguiente pueda comenzar.

```js
console.log("Inicio");
console.log("Tarea sincrónica");
console.log("Fin");
```

Salida:

```
Inicio
Tarea sincrónica
Fin
```

### Asincronía

Las operaciones asincrónicas permiten que el código continúe ejecutándose sin esperar a que ciertas tareas se completen, lo que optimiza el uso de recursos.

```js
console.log("Inicio");

setTimeout(() => {
  console.log("Tarea asincrónica");
}, 2000);

console.log("Fin");
```

Salida:

```
Inicio
Fin
Tarea asincrónica

```

En este caso, setTimeout() es una función asincrónica que introduce un retraso de 2 segundos antes de ejecutar su callback, pero el resto del código sigue ejecutándose mientras tanto.

## Mecanismos de Control Asincrónico en JavaScript

### Callbacks

Los callbacks son funciones que se pasan como argumentos a otras funciones y se ejecutan después de que se completa una operación asincrónica. Este fue el primer método utilizado para manejar asincronía en JavaScript, pero puede generar código difícil de leer y mantener, especialmente cuando hay múltiples operaciones anidadas, fenómeno conocido como "callback hell".

### Promesas

Las promesas fueron introducidas en ECMAScript 6 (ES6) para mejorar el manejo de la asincronía y hacer el código más legible y estructurado. Una promesa es un objeto que representa la eventual finalización (o falla) de una operación asincrónica. Una promesa puede estar en uno de estos tres estados:

- **Pending (pendiente):** La promesa está en curso.
- **Fulfilled (cumplida):** La operación se completó con éxito.
- **Rejected (rechazada):** La operación falló.

### async / await

ECMAScript 2017 introdujo las funciones `async/await` para simplificar aún más el manejo de promesas. La palabra clave `async` permite que una función devuelva una promesa, mientras que `await` detiene la ejecución dentro de una función `async` hasta que la promesa se resuelva o se rechace.
