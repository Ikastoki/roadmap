<h1 align='center'>setTimeout, setInterval y clear </h1>

<h2>📑 Contenido</h2>

- [setTimeout, setInterval y clear](#settimeout-setinterval-y-clear)
- [setTimeout()](#settimeout)
  - [Ejemplo básico](#ejemplo-básico)
  - [Con parámetros](#con-parámetros)
  - [Cancelar setTimeout:](#cancelar-settimeout)
- [setInterval()](#setinterval)
  - [Ejemplo básico](#ejemplo-básico-1)
  - [Con parámetros](#con-parámetros-1)
  - [Cancelar setInterval](#cancelar-setinterval)
- [Diferencias entre setTimeout y setInterval](#diferencias-entre-settimeout-y-setinterval)
- [Ejecución de código asíncrono y temporizadores](#ejecución-de-código-asíncrono-y-temporizadores)
- [Consideraciones de rendimiento y temporizadores](#consideraciones-de-rendimiento-y-temporizadores)

## setTimeout, setInterval y clear

En JavaScript, gestionar el tiempo y programar tareas que se ejecuten después de un periodo específico es fundamental en diversas aplicaciones. Las funciones `setTimeout` y `setInterval` son las herramientas principales para trabajar con temporizadores. Además, se utiliza `clearInterval` para detener la ejecución de los intervalos.

## setTimeout()

La función `setTimeout` permite ejecutar una función o fragmento de código después de un tiempo especificado en milisegundos.

### Ejemplo básico

```js
function saludar() {
  console.log("Hola, mundo!");
}

setTimeout(saludar, 2000); // Ejecuta la función después de 2 segundos.
```

### Con parámetros

Es posible pasar parámetros a la función que se ejecutará cuando el temporizador expire.

```js
function saludar(nombre) {
  console.log(`Hola, ${nombre}`);
}

setTimeout(saludar, 3000, "Ana"); // "Hola, Ana" se imprime después de 3 segundos.
```

### Cancelar setTimeout:

El método `clearTimeout` se usa para cancelar un setTimeout antes de que se ejecute.

```js
const temporizador = setTimeout(() => {
  console.log("Esto no se mostrará");
}, 5000);

clearTimeout(temporizador); // Cancela el temporizador.
```

En este caso, aunque `setTimeout` está configurado para ejecutarse después de 5 segundos, la función `clearTimeout(temporizador)` detiene su ejecución antes de que se complete.

## setInterval()

La función `setInterval` ejecuta una función o código repetidamente en intervalos de tiempo especificados (también en milisegundos). A diferencia de `setTimeout`, que se ejecuta solo una vez, `setInterval` continúa ejecutando la función cada cierto tiempo hasta que se cancele.

### Ejemplo básico

```js
function mostrarHora() {
  const ahora = new Date();
  console.log(ahora.toLocaleTimeString());
}

setInterval(mostrarHora, 1000); // Muestra la hora actual cada segundo.
```

### Con parámetros

Puedes pasar parámetros a la función que será ejecutada en cada intervalo.

```js
function saludar(nombre) {
  console.log(`Hola, ${nombre}`);
}

setInterval(saludar, 2000, "Ana"); // Muestra "Hola, Ana" cada 2 segundos.
```

### Cancelar setInterval

El método `clearInterval` se utiliza para detener la ejecución repetida de un setInterval.

```js
const intervalo = setInterval(() => {
  console.log("Esto se repetirá");
}, 1000);

setTimeout(() => {
  clearInterval(intervalo); // Detiene el intervalo después de 5 segundos.
}, 5000);
```

En este ejemplo, el `setInterval` se ejecuta cada segundo, pero el `setTimeout` detiene el intervalo después de 5 segundos, usando `clearInterval`.

> [!NOTE]
> Tanto `clearTimeout` como `clearInterval` se usan para cancelar temporizadores. Recuerda que ambos requieren que guardes el valor de retorno de `setTimeout` o `setInterval` en una variable para cancelarlos más tarde.

## Diferencias entre setTimeout y setInterval

Aunque ambos métodos están diseñados para ejecutar código con un retraso, la principal diferencia es que:

- **setTimeout:** Ejecuta la función solo una vez, después de un retraso.
- **setInterval:** Ejecuta la función repetidamente en intervalos de tiempo hasta que se detiene explícitamente con clearInterval.

## Ejecución de código asíncrono y temporizadores

El uso de `setTimeout` y `setInterval` tiene una conexión directa con la naturaleza asíncrona de JavaScript. Ambos métodos permiten programar la ejecución de funciones en el futuro, mientras el resto del código sigue ejecutándose.

```js
console.log("Inicio");

setTimeout(() => {
  console.log("Esto se ejecuta después");
}, 2000);

console.log("Fin");
```

En este ejemplo, aunque `setTimeout` está configurado para ejecutarse después de 2 segundos, el resto del código sigue ejecutándose. Por lo tanto, verás el siguiente resultado en la consola:

```
Inicio
Fin
Esto se ejecuta después
```

## Consideraciones de rendimiento y temporizadores

El uso excesivo de `setInterval` o `setTimeout` con tiempos muy pequeños puede afectar negativamente el rendimiento, especialmente si las funciones ejecutadas son intensivas en procesamiento o hay muchos temporizadores ejecutándose simultáneamente. En estos casos, es importante cancelar los temporizadores cuando ya no son necesarios, utilizando `clearTimeout` o `clearInterval`.
