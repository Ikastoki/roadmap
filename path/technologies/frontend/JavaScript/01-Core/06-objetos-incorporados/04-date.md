<h1 align='center'>Date</h1>

<h2>📑 Contenido</h2>

- [Date](#date)
- [Métodos Estáticos de Date](#métodos-estáticos-de-date)
  - [Date.now()](#datenow)
  - [Date.parse()](#dateparse)
  - [Date.UTC()](#dateutc)
- [Métodos de Obtención (Getters)](#métodos-de-obtención-getters)
  - [getDate()](#getdate)
  - [getDay()](#getday)
  - [getFullYear()](#getfullyear)
  - [getHours()](#gethours)
  - [getMilliseconds()](#getmilliseconds)
  - [getMinutes()](#getminutes)
  - [getMonth()](#getmonth)
  - [getSeconds()](#getseconds)
  - [getTime()](#gettime)
  - [getTimezoneOffset()](#gettimezoneoffset)
  - [getUTCDate()](#getutcdate)
  - [getUTCDay()](#getutcday)
  - [getUTCFullYear()](#getutcfullyear)
  - [getUTCHours()](#getutchours)
  - [getUTCMilliseconds()](#getutcmilliseconds)
  - [getUTCMinutes()](#getutcminutes)
  - [getUTCMonth()](#getutcmonth)
  - [getUTCSeconds()](#getutcseconds)
  - [getYear() (obsoleto)](#getyear-obsoleto)
- [Métodos de Establecimiento (Setters)](#métodos-de-establecimiento-setters)
  - [setDate()](#setdate)
  - [setFullYear()](#setfullyear)
  - [setHours()](#sethours)
  - [setMilliseconds()](#setmilliseconds)
  - [setMinutes()](#setminutes)
  - [setMonth()](#setmonth)
  - [setSeconds()](#setseconds)
  - [setTime()](#settime)
  - [setUTCDate()](#setutcdate)
  - [setUTCFullYear()](#setutcfullyear)
  - [setUTCHours()](#setutchours)
  - [setUTCMilliseconds()](#setutcmilliseconds)
  - [setUTCMinutes()](#setutcminutes)
  - [setUTCMonth()](#setutcmonth)
  - [setUTCSeconds()](#setutcseconds)
  - [setYear() (obsoleto)](#setyear-obsoleto)
- [Métodos de Conversión](#métodos-de-conversión)
  - [toDateString()](#todatestring)
  - [toISOString()](#toisostring)
  - [toJSON()](#tojson)
  - [toLocaleDateString()](#tolocaledatestring)
  - [toLocaleString()](#tolocalestring)
  - [toLocaleTimeString()](#tolocaletimestring)
  - [toString()](#tostring)
  - [toTimeString()](#totimestring)
  - [toUTCString()](#toutcstring)
- [Método Valor Primitivo](#método-valor-primitivo)
  - [valueOf()](#valueof)

## Date

El objeto `Date` en JavaScript es una poderosa herramienta para manejar fechas y tiempos, permitiendo crear, manipular y extraer información sobre fechas, horas, minutos, segundos, y más. En este capítulo, se abordarán los métodos estáticos y de instancia que ofrece el objeto `Date`, categorizados en métodos estáticos, métodos de obtención (getters), métodos de establecimiento (setters), métodos de conversión y otros métodos misceláneos.

## Métodos Estáticos de Date

Los métodos estáticos de `Date` permiten trabajar con fechas y horas sin necesidad de crear una instancia de `Date`.

### Date.now()

Devuelve el número de milisegundos transcurridos desde el 1 de enero de 1970 (Epoch).

```js
console.log(Date.now()); // Imprime: 1695224223445 (ejemplo)
```

### Date.parse()

Convierte una cadena de texto que representa una fecha (en formato legible) en milisegundos desde el 1 de enero de 1970.

```js
console.log(Date.parse("December 17, 1995 03:24:00")); // Imprime: 819199440000
```

### Date.UTC()

Devuelve el número de milisegundos desde el 1 de enero de 1970, pero interpretando los argumentos como tiempo UTC.

```js
console.log(Date.UTC(2023, 8, 20)); // Imprime: 1695168000000
```

## Métodos de Obtención (Getters)

Los métodos de obtención permiten extraer información de una instancia de `Date`, como el día, el mes, el año, la hora, etc. Existen versiones locales y UTC de estos métodos, que devuelven los valores basados en la hora local o en la hora universal (UTC).

### getDate()

Devuelve el día del mes (del 1 al 31).

```js
let fecha = new Date();
console.log(fecha.getDate()); // Imprime: 20 (ejemplo)
```

### getDay()

Devuelve el día de la semana (0 para domingo, 6 para sábado).

```js
console.log(fecha.getDay()); // Imprime: 3 (miércoles)
```

### getFullYear()

Devuelve el año completo.

```js
console.log(fecha.getFullYear()); // Imprime: 2023
```

### getHours()

Devuelve la hora (de 0 a 23).

```js
console.log(fecha.getHours()); // Imprime: 14 (ejemplo)
```

### getMilliseconds()

Devuelve los milisegundos (de 0 a 999).

```js
console.log(fecha.getMilliseconds()); // Imprime: 345 (ejemplo)
```

### getMinutes()

Devuelve los minutos (de 0 a 59).

```js
console.log(fecha.getMinutes()); // Imprime: 24 (ejemplo)
```

### getMonth()

Devuelve el mes (de 0 a 11, donde 0 representa enero y 11 representa diciembre).

```js
console.log(fecha.getMonth()); // Imprime: 8 (septiembre)
```

### getSeconds()

Devuelve los segundos (de 0 a 59).

```js
console.log(fecha.getSeconds()); // Imprime: 59 (ejemplo)
```

### getTime()

Devuelve el número de milisegundos desde el 1 de enero de 1970.

```js
console.log(fecha.getTime()); // Imprime: 1695224223445 (ejemplo)
```

### getTimezoneOffset()

Devuelve la diferencia en minutos entre la hora UTC y la hora local.

```js
console.log(fecha.getTimezoneOffset()); // Imprime: -120 (ejemplo, para GMT+2)
```

### getUTCDate()

Devuelve el día del mes en UTC.

```js
console.log(fecha.getUTCDate()); // Imprime: 20
```

### getUTCDay()

Devuelve el día de la semana en UTC (0 para domingo, 6 para sábado).

```js
console.log(fecha.getUTCDay()); // Imprime: 3 (miércoles)
```

### getUTCFullYear()

Devuelve el año completo en UTC.

```js
console.log(fecha.getUTCFullYear()); // Imprime: 2023
```

### getUTCHours()

Devuelve la hora en UTC.

```js
console.log(fecha.getUTCHours()); // Imprime: 12 (ejemplo)
```

### getUTCMilliseconds()

Devuelve los milisegundos en UTC.

```js
console.log(fecha.getUTCMilliseconds()); // Imprime: 345 (ejemplo)
```

### getUTCMinutes()

Devuelve los minutos en UTC.

```js
console.log(fecha.getUTCMinutes()); // Imprime: 24 (ejemplo)
```

### getUTCMonth()

Devuelve el mes en UTC (de 0 a 11).

```js
console.log(fecha.getUTCMonth()); // Imprime: 8 (septiembre)
```

### getUTCSeconds()

Devuelve los segundos en UTC.

```js
console.log(fecha.getUTCSeconds()); // Imprime: 59 (ejemplo)
```

### getYear() (obsoleto)

Este método está obsoleto y no debería ser usado. Devuelve el año menos 1900.

```js
console.log(fecha.getYear()); // Imprime: 123 (2023 - 1900)
```

## Métodos de Establecimiento (Setters)

Los métodos de establecimiento permiten modificar la fecha y la hora de una instancia `Date`. Al igual que con los getters, algunos de estos métodos también tienen versiones UTC.

### setDate()

Establece el día del mes.

```js
fecha.setDate(25);
console.log(fecha.getDate()); // Imprime: 25
```

### setFullYear()

Establece el año completo.

```js
fecha.setFullYear(2025);
console.log(fecha.getFullYear()); // Imprime: 2025
```

### setHours()

Establece la hora (de 0 a 23).

```js
fecha.setHours(18);
console.log(fecha.getHours()); // Imprime: 18
```

### setMilliseconds()

Establece los milisegundos.

```js
fecha.setMilliseconds(500);
console.log(fecha.getMilliseconds()); // Imprime: 500
```

### setMinutes()

Establece los minutos.

```js
fecha.setMinutes(45);
console.log(fecha.getMinutes()); // Imprime: 45
```

### setMonth()

Establece el mes (de 0 a 11).

```js
fecha.setMonth(11);
console.log(fecha.getMonth()); // Imprime: 11 (diciembre)
```

### setSeconds()

Establece los segundos.

```js
fecha.setSeconds(30);
console.log(fecha.getSeconds()); // Imprime: 30
```

### setTime()

Establece el valor de la fecha en milisegundos desde el 1 de enero de 1970.

```js
fecha.setTime(819199440000);
console.log(fecha.toISOString()); // Imprime: 1995-12-17T03:24:00.000Z
```

### setUTCDate()

Establece el día del mes en UTC.

```js
fecha.setUTCDate(15);
console.log(fecha.getUTCDate()); // Imprime: 15
```

### setUTCFullYear()

Establece el año completo en UTC.

```js
fecha.setUTCFullYear(2020);
console.log(fecha.getUTCFullYear()); // Imprime: 2020
```

### setUTCHours()

Establece la hora en UTC.

```js
fecha.setUTCHours(12);
console.log(fecha.getUTCHours()); // Imprime: 12
```

### setUTCMilliseconds()

Establece los milisegundos en UTC.

```js
fecha.setUTCMilliseconds(200);
console.log(fecha.getUTCMilliseconds()); // Imprime: 200
```

### setUTCMinutes()

Establece los minutos en UTC.

```js
fecha.setUTCMinutes(30);
console.log(fecha.getUTCMinutes()); // Imprime: 30
```

### setUTCMonth()

Establece el mes en UTC (de 0 a 11).

```js
fecha.setUTCMonth(1); // Establece febrero (mes 1)
console.log(fecha.getUTCMonth()); // Imprime: 1
```

### setUTCSeconds()

Establece los segundos en UTC.

```js
fecha.setUTCSeconds(50);
console.log(fecha.getUTCSeconds()); // Imprime: 50
```

### setYear() (obsoleto)

Establece el año (menos 1900). Este método está obsoleto y no debería ser utilizado.

```js
fecha.setYear(99); // Establece 1999
console.log(fecha.getFullYear()); // Imprime: 1999
```

## Métodos de Conversión

Los métodos de conversión permiten transformar una instancia de `Date` en cadenas de texto con distintos formatos.

### toDateString()

Devuelve la parte de la fecha en formato legible (sin la hora).

```js
console.log(fecha.toDateString()); // Imprime: "Wed Sep 20 2023"
```

### toISOString()

Devuelve la fecha en formato ISO 8601.

```js
console.log(fecha.toISOString()); // Imprime: "2023-09-20T12:24:59.345Z"
```

### toJSON()

Devuelve la fecha en formato JSON (que es idéntico al formato ISO).

```js
console.log(fecha.toJSON()); // Imprime: "2023-09-20T12:24:59.345Z"
```

### toLocaleDateString()

Devuelve la fecha en formato de cadena según la configuración regional (locale) del sistema.

```js
console.log(fecha.toLocaleDateString()); // Imprime: "9/20/2023" (ejemplo en configuración de EE.UU.)
```

### toLocaleString()

Devuelve la fecha y la hora en formato de cadena según la configuración regional.

```js
console.log(fecha.toLocaleString()); // Imprime: "9/20/2023, 12:24:59 PM"
```

### toLocaleTimeString()

Devuelve solo la hora en formato de cadena según la configuración regional.

```js
console.log(fecha.toLocaleTimeString()); // Imprime: "12:24:59 PM"
```

### toString()

Devuelve la fecha completa como una cadena de texto en formato legible.

```js
console.log(fecha.toString()); // Imprime: "Wed Sep 20 2023 14:24:59 GMT+0200 (Central European Summer Time)"
```

### toTimeString()

Devuelve solo la parte de la hora en formato legible.

```js
console.log(fecha.toTimeString()); // Imprime: "14:24:59 GMT+0200 (Central European Summer Time)"
```

### toUTCString()

Devuelve la fecha completa como una cadena de texto en formato UTC.

```js
console.log(fecha.toUTCString()); // Imprime: "Wed, 20 Sep 2023 12:24:59 GMT"
```

## Método Valor Primitivo

### valueOf()

Devuelve el valor primitivo de un objeto Date, que es el número de milisegundos desde el 1 de enero de 1970.

```js
console.log(fecha.valueOf()); // Imprime: 1695224223445
```
