<h1 align="center">Afirmaciones (Assertions)</h1>

<h2>📑 Contenido</h2>

- [Afirmaciones (Assertions)](#afirmaciones-assertions)
- [as const](#as-const)
- [as\[type\]](#astype)
- [as any](#as-any)
- [Non-Null Satisfies](#non-null-satisfies)

## Afirmaciones (Assertions)

Las "assertions" (afirmaciones) son una forma de indicar al compilador que confíe en el desarrollador en cuanto al tipo de una variable o expresión, incluso si el compilador no puede inferirlo de forma automática. Las "assertions" se utilizan con el operador as.

## as const

Se utiliza para marcar una variable como de tipo constante. Esto significa que TypeScript inferirá un tipo literal para la variable, en lugar de un tipo más amplio basado en el valor asignado.

```ts

```

## as[type]

Se utiliza para realizar una conversión de tipo explícita. Puedes usar cualquier tipo después de as, pero debes tener en cuenta que si el tipo real del valor no coincide con el tipo especificado, TypeScript no realizará ninguna verificación en tiempo de ejecución.

```ts
const x = 10; // TypeScript infiere el tipo como number
const y = 10 as const; // TypeScript infiere el tipo como 10 (literal)
```

## as any

Se utiliza para forzar el tipo de una variable o expresión a any. Esto desactiva la comprobación de tipos de TypeScript para esa variable o expresión, lo que significa que puedes realizar cualquier operación sobre ella sin obtener errores de tipo.

```ts
let mensaje: any = "Hola";
let longitud = (mensaje as string).length; // Conversión a tipo string
```

## Non-Null Satisfies

No estoy seguro de qué estás preguntando específicamente con este término. En TypeScript, el operador de acceso a miembros ! se puede utilizar para indicarle al compilador que confíe en que una expresión no es null o undefined.

```ts
let elemento: HTMLElement | null = document.getElementById("miElemento");
elemento!.classList.add("activo"); // TypeScript confía en que elemento no es null
```
