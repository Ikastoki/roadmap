<h1 align="center">Operators</h1>

<h2>📑 Contenido</h2>

- [Operators](#operators)
- [Operadores Aritméticos](#operadores-aritméticos)
- [Operadores de Comparación](#operadores-de-comparación)
- [Operadores Lógicos](#operadores-lógicos)

## Operators

Los operadores en Sass te permiten realizar operaciones matemáticas, lógicas y de comparación directamente en tus hojas de estilo. Esto es increíblemente útil para crear estilos dinámicos, calcular tamaños y espaciados, y hacer tu código más inteligente.

## Operadores Aritméticos

Sass puede realizar operaciones matemáticas básicas como suma (+), resta (-), multiplicación (\*), división (/) y módulo (%).

```scss
// Sass
$ancho-base: 100px;
$padding: 20px;

.caja-grande {
  width: $ancho-base * 2; // 200px
  height: $ancho-base + 50px; // 150px
  padding: $padding / 2; // 10px
  margin-left: 200px % 30px; // 20px (el resto de la división)
}
```

> [!IMPORTANT]
>
> Importante sobre la división:
> Sass interpreta el símbolo de división (/) de dos maneras:
>
> - Como una división matemática, si la expresión está dentro de paréntesis, es parte de otra operación, o si se usa en conjunción con una variable.
>
> - Como un separador de valores, como en font: 16px/1.5 Arial;.

Si quieres asegurarte de que Sass realice una división, usa paréntesis o incluye al menos una variable en la operación.

```scss
// Sass
// Esto es una división
$ancho-mitad: 100px / 2;
.clase-con-division {
  width: (100px / 2);
}

// Esto es un separador de valores
.font-size-and-line-height {
  font: 16px/1.5 sans-serif;
}
```

## Operadores de Comparación

Puedes comparar valores usando:

- Igual a (==)

- Distinto de (!=)

- Mayor que (>)

- Mayor o igual que (>=)

- Menor que (<)

- Menor o igual que (<=)

Estos operadores son muy útiles en directivas condicionales como `@if`.

```scss
// Sass
$tamanio-fuente: 16px;

@if $tamanio-fuente > 14px {
  body {
    font-size: 1.2rem;
  }
}
```

## Operadores Lógicos

Sass soporta los operadores lógicos `and`, `or` y `not`.

- `and`: Devuelve true si ambas expresiones son verdaderas.

- `or`: Devuelve true si al menos una de las expresiones es verdadera.

- `not`: Invierte el valor booleano de una expresión.

```scss
// Sass
$modo-oscuro: true;
$usuario-premium: false;

.tarjeta-de-usuario {
  @if $modo-oscuro and $usuario-premium {
    border: 2px solid gold;
  } @else if $modo-oscuro or $usuario-premium {
    border: 1px solid #ccc;
  }
}
```

En este caso, la tarjeta solo tendrá un borde dorado si se cumplen ambas condiciones.
