<h1 align='center'>El Ciclo de Eventos y la Concurrencia </h1>

<h2>📑 Contenido</h2>

- [El Ciclo de Eventos y la Concurrencia](#el-ciclo-de-eventos-y-la-concurrencia)
- [Concurrencia en JavaScript](#concurrencia-en-javascript)
- [¿Qué es el Event Loop?](#qué-es-el-event-loop)
- [Funcionamiento del Event Loop](#funcionamiento-del-event-loop)
- [Detalles del Event Loop: Fases](#detalles-del-event-loop-fases)
- [Ejemplos de Event Loop en Acción](#ejemplos-de-event-loop-en-acción)
  - [Operaciones Sincrónicas y Asincrónicas](#operaciones-sincrónicas-y-asincrónicas)
  - [Callback en I/O](#callback-en-io)
- [Microtareas vs Macrotareas](#microtareas-vs-macrotareas)
- [Errores Comunes y Buenas Prácticas](#errores-comunes-y-buenas-prácticas)
  - [Bloqueo del Event Loop](#bloqueo-del-event-loop)
  - [Buen uso de Microtareas](#buen-uso-de-microtareas)

## El Ciclo de Eventos y la Concurrencia

JavaScript es conocido por ser un lenguaje monohilo, lo que significa que solo puede ejecutar una tarea a la vez en el hilo principal. Sin embargo, gracias a su arquitectura basada en el Event Loop (ciclo de eventos), puede gestionar operaciones asincrónicas de manera eficiente, permitiendo que el código siga funcionando mientras se procesan tareas que pueden tardar en completarse, como solicitudes de red o temporizadores.

En este capítulo, exploraremos cómo funciona el Ciclo de Eventos en JavaScript, el concepto de concurrencia y cómo interactúan los diferentes componentes para permitir la ejecución fluida de tareas asincrónicas.

## Concurrencia en JavaScript

JavaScript es concurrente debido a su capacidad para manejar múltiples tareas "al mismo tiempo", pero no en el sentido tradicional de la multitarea paralela. En lugar de ejecutar tareas simultáneamente en diferentes hilos, JavaScript usa un modelo de concurrencia basado en eventos. El modelo depende del Event Loop para gestionar las tareas asincrónicas sin bloquear el flujo del programa.

Aunque JavaScript es monohilo, puede delegar algunas operaciones (como E/S de archivos, solicitudes HTTP, etc.) a APIs del navegador o al sistema operativo. Estas tareas se ejecutan en segundo plano, mientras el hilo principal sigue manejando otras operaciones.

## ¿Qué es el Event Loop?

El Event Loop es el mecanismo que permite que JavaScript maneje la ejecución de código asincrónico. Coordina la ejecución de funciones, la colocación de tareas en cola y la administración de la pila de llamadas. En términos simples, se asegura de que las operaciones asincrónicas se manejen sin bloquear la ejecución del código.

El Event Loop se compone de tres elementos principales:

1. **Call Stack (Pila de llamadas):** Una estructura LIFO (Last In, First Out) donde se almacenan las funciones a medida que son invocadas.
1. **Task Queue (Cola de tareas):** Una lista de funciones (callbacks) que están esperando para ser ejecutadas una vez que la pila de llamadas esté vacía.
1. **Event Loop:** Monitorea continuamente la pila de llamadas y la cola de tareas. Si la pila de llamadas está vacía, toma una tarea de la cola de tareas y la coloca en la pila para ejecutarla.

## Funcionamiento del Event Loop

Veamos paso a paso cómo funciona el Event Loop en la práctica:

1. El Call Stack (Pila de Llamadas) comienza vacío.
   Cuando una función es invocada, se coloca en la pila de llamadas.
1. Si la función realiza una operación asincrónica (por ejemplo, una solicitud de red), esta tarea es delegada a una API del navegador (o al entorno de ejecución como Node.js) para manejarla en segundo plano.
1. Mientras la operación asincrónica está en progreso, la función continúa su ejecución, o si está esperando, la ejecución continúa con otras tareas.
1. Una vez que la operación asincrónica se completa (por ejemplo, la solicitud HTTP recibe una respuesta), se coloca un callback en la cola de tareas (Task Queue).
1. El Event Loop revisa la pila de llamadas. Si la pila está vacía, toma la siguiente tarea de la cola y la coloca en la pila para que se ejecute.

## Detalles del Event Loop: Fases

El Event Loop está compuesto por varias fases, cada una de las cuales se encarga de diferentes tipos de tareas. En el entorno de Node.js, las fases del Event Loop se pueden desglosar así:

1. **Timers:** Ejecuta los callbacks de funciones como setTimeout() y setInterval() cuyos temporizadores han expirado.

1. **I/O Callbacks:** Procesa callbacks de operaciones de E/S (Entrada/Salida) completadas, como la lectura de archivos o la finalización de una solicitud HTTP.

1. **Idle, Prepare:** Una fase interna del Event Loop usada por el sistema.

1. **Poll:** Esta fase recupera eventos pendientes de E/S y ejecuta sus callbacks. Si no hay nada pendiente, el Event Loop puede detenerse y esperar por nuevas tareas.

1. **Check:** Aquí se ejecutan los callbacks de setImmediate(), que se colocan al final del ciclo del Event Loop.

1. **Close Callbacks:** Ejecuta callbacks de eventos de cierre, como socket.on('close').

## Ejemplos de Event Loop en Acción

### Operaciones Sincrónicas y Asincrónicas

```js
console.log("Inicio"); // Operación sincrónica

setTimeout(() => {
  console.log("Timeout"); // Operación asincrónica
}, 2000);

console.log("Fin"); // Operación sincrónica
```

Salida:

```
Inicio
Fin
Timeout
```

En este ejemplo, el `setTimeout()` es una operación asincrónica que coloca el callback en la cola de tareas después de 2 segundos. Mientras tanto, el Event Loop sigue ejecutando el resto del código sin esperar a que la operación se complete.

### Callback en I/O

```js
const fs = require("fs");

console.log("Inicio de la lectura");

fs.readFile("archivo.txt", "utf-8", (err, data) => {
  if (err) throw err;
  console.log("Lectura completada");
});

console.log("Fin del código");
```

```
Inicio de la lectura
Fin del código
Lectura completada
```

En este caso, la operación de lectura de archivo se delega al sistema, y el Event Loop sigue ejecutando el código mientras tanto. Una vez que el archivo ha sido leído, el callback se coloca en la cola de tareas y el Event Loop lo ejecuta cuando la pila de llamadas está vacía.

## Microtareas vs Macrotareas

En el Event Loop, las tareas se pueden clasificar en dos tipos: Macrotareas y Microtareas. La diferencia clave es cuándo son ejecutadas en el ciclo de eventos.

- **Macrotareas:** Incluyen eventos como los callbacks de `setTimeout`, `setInterval`, y operaciones de I/O.
- **Microtareas:** Son tareas de mayor prioridad, como las manejadas por las promesas (callbacks de `.then()`). Se ejecutan inmediatamente después de que la pila de llamadas esté vacía, antes de que se procesen las macrotareas.

```js
console.log("Inicio");

setTimeout(() => {
  console.log("Timeout"); // Macrotarea
}, 0);

Promise.resolve().then(() => {
  console.log("Promise"); // Microtarea
});

console.log("Fin");
```

Salida:

```
Inicio
Fin
Promise
Timeout
```

Aunque `setTimeout()` está programado para ejecutarse después de 0 milisegundos, las microtareas (en este caso, la promesa) se ejecutan antes que las macrotareas.

## Errores Comunes y Buenas Prácticas

### Bloqueo del Event Loop

Aunque JavaScript puede manejar múltiples tareas asincrónicas, bloquear el Event Loop con operaciones sincrónicas largas puede causar problemas en la experiencia del usuario. Por ejemplo:

```js
// Bucle pesado que bloquea el Event Loop
for (let i = 0; i < 1e9; i++) {}

console.log("Este mensaje tardará en aparecer");
```

Evita operaciones que consuman mucho tiempo en el hilo principal. Si es necesario realizar cálculos intensivos, considera el uso de Web Workers en el navegador o child processes en Node.js para mover estas tareas fuera del hilo principal.

### Buen uso de Microtareas

Es crucial usar promesas y microtareas adecuadamente para evitar el desbordamiento del Event Loop. Un exceso de microtareas puede retrasar la ejecución de las macrotareas.
