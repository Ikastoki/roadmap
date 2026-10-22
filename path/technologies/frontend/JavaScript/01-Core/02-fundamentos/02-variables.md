<h1 align='center'>Variables</h1>

<h2>📑 Contenido</h2>

- [Variables](#variables)
- [¿Qué es una variable?](#qué-es-una-variable)
- [¿Cómo funcionan las variables en JavaScript?](#cómo-funcionan-las-variables-en-javascript)
- [Reglas para nombrar variables](#reglas-para-nombrar-variables)
- [Tipos de variables](#tipos-de-variables)
  - [Let](#let)
  - [Const](#const)
  - [Var](#var)

## Variables

En este capítulo, exploraremos un concepto fundamental en la programación: las variables. Aprenderemos qué son, para qué sirven, cómo utilizarlas y las distintas formas en que se pueden declarar en JavaScript.

## ¿Qué es una variable?

Imagina que una variable es como una caja con una etiqueta. En esa caja, puedes guardar cosas. Luego, cuando necesites usar esa cosa (un número, un nombre, una palabra, etc.), solo tienes que llamar por la etiqueta para sacar lo que está dentro.

En programación, una variable guarda información que luego puedes usar o cambiar más tarde.

Ejemplo del mundo real:

Pensemos en una caja con una etiqueta que dice "edad". Dentro de esa caja, puedes poner el número de años que tienes, por ejemplo, 25. Si en algún momento cumples años, solo cambias lo que está dentro de la caja sin cambiar la etiqueta.

## ¿Cómo funcionan las variables en JavaScript?

En JavaScript, usar una variable es como decirle al computador: "Guarda esto para mí bajo este nombre, para que pueda usarlo después". Veamos cómo se hace.

1. **Declarar una variable:**

   Para decirle a JavaScript que quieres crear una variable, usamos la palabra let o const seguida de un nombre.

   ```js
   let nombre; // Aquí le estamos diciendo a JavaScript que tenemos una variable llamada "nombre".
   ```

   En este momento, hemos creado una "caja" llamada nombre, pero todavía no hemos puesto nada dentro. La caja está vacía.

2. **Asignar un valor a una variable:**

   Ahora que tenemos la caja (la variable), podemos poner algo dentro. Para eso usamos el símbolo =. Esto se llama "asignar" un valor a la variable.

   ```js
   nombre = "Carlos"; // Aquí estamos guardando el nombre "Carlos" dentro de la caja llamada "nombre".
   ```

   Así, la caja nombre ahora tiene adentro la palabra "Carlos". Cada vez que usemos nombre, JavaScript sabrá que estamos hablando de "Carlos".

   Podemos hacer estas dos cosas (declarar y asignar) en una sola línea

   ```js
   let nombre = "Carlos"; // Declaramos la variable y le asignamos un valor al mismo tiempo.
   ```

3. **Usar una variable:**

   Una vez que has guardado algo en una variable, puedes usarlo en cualquier parte de tu código.

   ```js
   let nombre = "Carlos";
   console.log(nombre); // Esto imprimirá "Carlos" en la consola.
   ```

   Aquí, console.log es una función que muestra en la pantalla lo que pongas entre paréntesis. Como dentro de los paréntesis está la variable nombre, se mostrará su contenido, que es "Carlos".

4. Cambiar el valor de una variable:

   Si en algún momento necesitas cambiar lo que está guardado en una variable, puedes hacerlo simplemente asignando un nuevo valor.

   ```js
   let nombre = "Carlos"; // Primero, guardamos "Carlos" en la variable.
   nombre = "Ana"; // Ahora cambiamos el contenido de la variable a "Ana".
   console.log(nombre); // Esto imprimirá "Ana".
   ```

## Reglas para nombrar variables

En JavaScript, hay algunas reglas que debes seguir al elegir un nombre para una variable:

1.  El nombre debe comenzar con una letra, un signo de dólar $, o un guion bajo \_. No puede comenzar con un número.

    - Ejemplo válido: edad
    - Ejemplo no válido: 3amigos

2.  Los nombres de las variables no pueden contener espacios. Si quieres usar varias palabras, puedes usar una convención llamada camelCase, donde la primera palabra va en minúsculas y cada palabra siguiente comienza con mayúsculas.

    - Ejemplo: nombreCompleto

JavaScript distingue entre mayúsculas y minúsculas. Es decir, Nombre y nombre son dos variables diferentes.

## Tipos de variables

En JavaScript, hay diferentes formas de declarar variables let, const y var.

### Let

Utiliza let cuando quieras que el valor de la variable pueda cambiar en el futuro.
Es el más común cuando necesitas actualizar la información.

```js
let edad = 25;
edad = 26; // Aquí hemos cambiado el valor de "edad".
```

### Const

Utiliza const cuando sabes que el valor de la variable no va a cambiar nunca.
Es útil cuando estás seguro de que la información se mantendrá igual durante todo el programa.

```js
const pi = 3.1416; // El valor de "pi" no va a cambiar nunca.
```

> [!NOTE]
> Si intentas cambiar el valor de una variable declarada con const, JavaScript te dará un error.

### Var

var es otra manera de declarar variables, pero se usaba más en versiones antiguas de JavaScript. Hoy en día, es preferible usar let y const porque ofrecen un mejor control del código.
