<h1 align='center'>Expresiones y Declaraciones</h1>

<h2>📑 Contenido</h2>

- [Expresiones y Declaraciones](#expresiones-y-declaraciones)
- [¿Qué es una expresión?](#qué-es-una-expresión)
- [¿Qué es una declaración?](#qué-es-una-declaración)
- [Relación entre expresiones y declaraciones](#relación-entre-expresiones-y-declaraciones)
- [Llevadas a JS](#llevadas-a-js)
  - [Expresiones](#expresiones)
  - [Declaraciones](#declaraciones)

## Expresiones y Declaraciones

Voy a intentar explicar el tema de expresiones y declaraciones de una manera más básica, sin usar conceptos como variables, tipos de datos o funciones.

## ¿Qué es una expresión?

Imagina que estás haciendo una cuenta o una operación sencilla, como sumar dos números o comparar cosas. Eso es una expresión.

Una expresión es algo que **al final te da un valor o una respuesta**:

- Si piensas en el número 5, es una expresión porque tiene un valor (el valor es 5).
- Si haces una operación como 2 + 3, también es una expresión. El resultado de esa expresión es 5.
- Incluso algo como preguntar si hoy hace más calor que ayer es una expresión, porque al final obtienes una respuesta (por ejemplo, "sí" o "no").

## ¿Qué es una declaración?

Ahora, imagina que estás dando una instrucción o una orden. Una declaración es como decirle a alguien que haga algo o que tome una decisión.

Una declaración es una instrucción que le dice al programa **que haga algo**:

- Si le dices a alguien: "Si está lloviendo, usa paraguas", esa es una declaración. Le estás dando una **orden que depende de algo** (si está lloviendo o no).
- Otro ejemplo: si estás en clase y el maestro dice "Escribe tu nombre", eso también es una declaración, porque es una **instrucción**.

## Relación entre expresiones y declaraciones

Que une a las expresiones y declaraciones, pues que **una declaración puede tener expresiones dentro de ella**.

Ejemplo:

- Expresión: 2 + 3 (esto es una suma, te da el valor 5).
- Declaración: "Si el resultado de la suma es mayor que 4, di '¡Es un número grande!'".

Aquí estamos usando una expresión dentro de una declaración. Primero hacemos una cuenta (la expresión), y luego, dependiendo de esa cuenta, decidimos qué hacer (la declaración).

## Llevadas a JS

Intentemos trasladar los conceptos a JavaScript, si todavía no conoces los temas de la variables, tipos de datos, funciones etc... Por lo menos que te vaya sonando.

### Expresiones

Una expresión en JavaScript es cualquier fragmento de código que puede ser evaluado para devolver un valor. Pueden ser tan simples como un número o una operación matemática, o tan complejas como llamadas a funciones.

```js
5; // Número literal
x; // Una variable
x + 10; // Una operación aritmética
miFuncion(); // Una llamada a una función
true || false; // Una operación lógica
```

Como ya hemos visto, una expresión devuelve siempre un valor.

### Declaraciones

Una declaración (o sentencia) es una instrucción completa que realiza una acción. Las declaraciones pueden contener expresiones dentro de ellas, pero su principal función es controlar el flujo del programa o realizar tareas específicas.

Ejemplos:

- Declaración de variables
- Declaraciones condicionales
- Bucles (Loops)
- Declaración de funciones

```js
// Declaración de variable con una expresión (5)
let x = 5;

// Declaración de un condicional
if (x > 0) {
  console.log("x es positivo");
}

// Declaración de Bucles
for (let i = 0; i < 5; i++) {
  console.log(i);
}

// Declaración de una función
function suma(a, b) {
  return a + b;
}
```

> [!IMPORTANT]
>
> - **Expresión:** Siempre devuelve un valor.
> - **Declaración:** Realiza una acción o serie de acciones, y no necesariamente devuelve un valor directamente.
