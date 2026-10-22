<h1 align='center'>Desestructuración </h1>

<h2>📑 Contenido</h2>

- [Desestructuración](#desestructuración)
- [¿Qué es la Desestructuración?](#qué-es-la-desestructuración)
- [Desestructuración de Arrays](#desestructuración-de-arrays)
  - [Desestructuración básica de arrays](#desestructuración-básica-de-arrays)
  - [Omitir elementos en la desestructuración](#omitir-elementos-en-la-desestructuración)
  - [Valores predeterminados](#valores-predeterminados)
  - [Uso de desestructuración con el operador de rest](#uso-de-desestructuración-con-el-operador-de-rest)
- [Desestructuración de Objetos](#desestructuración-de-objetos)
  - [Desestructuración básica de objetos](#desestructuración-básica-de-objetos)
  - [Asignación a variables con nombres diferentes](#asignación-a-variables-con-nombres-diferentes)
  - [Valores predeterminados en objetos](#valores-predeterminados-en-objetos)
  - [Desestructuración de objetos anidados](#desestructuración-de-objetos-anidados)
  - [Uso del operador rest con objetos](#uso-del-operador-rest-con-objetos)
- [Desestructuración en Funciones](#desestructuración-en-funciones)
  - [Desestructuración de parámetros de arrays](#desestructuración-de-parámetros-de-arrays)
  - [Desestructuración de parámetros de objetos](#desestructuración-de-parámetros-de-objetos)
- [Usos Comunes y Beneficios](#usos-comunes-y-beneficios)

## Desestructuración

La desestructuración es una característica de JavaScript que te permite extraer valores de arrays u obtener propiedades de objetos y asignarlos a variables de una manera más limpia y compacta. Esta técnica es extremadamente útil para escribir código más conciso y mejorar la legibilidad, sobre todo cuando trabajas con datos complejos.

## ¿Qué es la Desestructuración?

La desestructuración es una forma de descomponer arrays y objetos en variables individuales mediante una sintaxis especial. En lugar de acceder a elementos de un array u objeto manualmente y asignarlos a variables una por una, puedes usar la desestructuración para hacer esto en una sola línea de código.

**Tipos de desestructuración:**

- Desestructuración de arrays.
- Desestructuración de objetos.

## Desestructuración de Arrays

Cuando trabajas con arrays, puedes extraer valores individuales y asignarlos a variables usando la sintaxis de desestructuración de arrays

### Desestructuración básica de arrays

```js
let numeros = [10, 20, 30];
let [a, b, c] = numeros;

console.log(a); // 10
console.log(b); // 20
console.log(c); // 30
```

En este caso, los valores 10, 20 y 30 del array numeros se asignan respectivamente a las variables a, b, y c.

### Omitir elementos en la desestructuración

Puedes omitir elementos de un array utilizando comas adicionales.

```js
let numeros = [10, 20, 30, 40];
let [a, , c] = numeros;

console.log(a); // 10
console.log(c); // 30
```

Aquí se omite el valor 20 al dejar un espacio vacío entre las comas.

### Valores predeterminados

Si el array no tiene suficientes elementos, puedes asignar valores predeterminados.

```js
let numeros = [10];
let [a, b = 20] = numeros;

console.log(a); // 10
console.log(b); // 20 (valor predeterminado)
```

En este ejemplo, como el segundo valor del array no existe, b toma el valor predeterminado 20.

### Uso de desestructuración con el operador de rest

El operador rest (`...`) puede ser utilizado para capturar el resto de los elementos en un array.

```js
let numeros = [10, 20, 30, 40];
let [a, ...resto] = numeros;

console.log(a); // 10
console.log(resto); // [20, 30, 40]
```

En este caso, la variable resto contiene todos los valores restantes del array después del primer elemento.

## Desestructuración de Objetos

La desestructuración de objetos te permite extraer propiedades de un objeto y asignarlas a variables. En lugar de acceder a las propiedades con notación de punto, puedes hacerlo directamente en una sola línea.

### Desestructuración básica de objetos

```js
let persona = { nombre: "Juan", edad: 30 };
let { nombre, edad } = persona;

console.log(nombre); // "Juan"
console.log(edad); // 30
```

En este ejemplo, las propiedades nombre y edad del objeto persona se asignan a las variables nombre y edad.

### Asignación a variables con nombres diferentes

Puedes asignar propiedades del objeto a variables con nombres diferentes utilizando la siguiente sintaxis:

```js
let persona = { nombre: "Juan", edad: 30 };
let { nombre: nombrePersona, edad: edadPersona } = persona;

console.log(nombrePersona); // "Juan"
console.log(edadPersona); // 30
```

Aquí, la propiedad nombre se asigna a nombrePersona y edad a edadPersona.

### Valores predeterminados en objetos

Al igual que con arrays, puedes proporcionar valores predeterminados si una propiedad no está presente en el objeto.

```js
let persona = { nombre: "Juan" };
let { nombre, edad = 25 } = persona;

console.log(nombre); // "Juan"
console.log(edad); // 25 (valor predeterminado)
```

Dado que edad no existe en el objeto, se le asigna el valor predeterminado 25.

### Desestructuración de objetos anidados

Puedes desestructurar objetos anidados fácilmente.

```js
let persona = {
  nombre: "Juan",
  direccion: { ciudad: "Madrid", pais: "España" },
};
let {
  nombre,
  direccion: { ciudad, pais },
} = persona;

console.log(nombre); // "Juan"
console.log(ciudad); // "Madrid"
console.log(pais); // "España"
```

En este caso, además de desestructurar nombre, también accedemos a las propiedades ciudad y país del objeto dirección dentro de persona.

### Uso del operador rest con objetos

Puedes usar el operador rest (...) para agrupar las propiedades restantes de un objeto en una variable.

```js
let persona = { nombre: "Juan", edad: 30, profesion: "Ingeniero" };
let { nombre, ...resto } = persona;

console.log(nombre); // "Juan"
console.log(resto); // { edad: 30, profesion: "Ingeniero" }
```

Aquí, la variable resto contiene las propiedades restantes (edad y profesion) después de desestructurar nombre.

## Desestructuración en Funciones

La desestructuración también es muy útil al trabajar con parámetros de funciones. En lugar de pasar y acceder a los elementos de arrays u objetos de manera tradicional, puedes desestructurarlos directamente en los parámetros de una función.

### Desestructuración de parámetros de arrays

```js
function imprimirValores([a, b, c]) {
  console.log(a, b, c);
}

let numeros = [10, 20, 30];
imprimirValores(numeros); // 10 20 30
```

### Desestructuración de parámetros de objetos

```js
function saludar({ nombre, edad }) {
  console.log(`Hola, mi nombre es ${nombre} y tengo ${edad} años.`);
}

let persona = { nombre: "Ana", edad: 25 };
saludar(persona); // Hola, mi nombre es Ana y tengo 25 años.
```

En este ejemplo, los parámetros nombre y edad se desestructuran directamente desde el objeto que se pasa como argumento.

## Usos Comunes y Beneficios

- **Código más limpio y legible:** La desestructuración reduce la cantidad de líneas necesarias para acceder a valores en arrays y objetos.
- **Asignación múltiple en una sola línea:** Puedes extraer múltiples valores o propiedades simultáneamente, haciendo el código más conciso.
- **Manejo de objetos grandes:** Permite acceder solo a las propiedades que necesitas sin afectar el objeto completo.
- **Funciona con cualquier estructura de datos:** La desestructuración es muy útil cuando se trata de manejar respuestas de APIs o datos anidados.
