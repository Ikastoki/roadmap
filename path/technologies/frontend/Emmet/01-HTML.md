<h1 align="center">EMMET-HTML</h1>

<h2>Contenido:</h2>

- [Sintaxis](#sintaxis)
- [Operadores principales:](#operadores-principales)
  - [Operador Hijo: \>](#operador-hijo-)
  - [Operador Hermano: +](#operador-hermano-)
  - [Operador Multiplicar: \*](#operador-multiplicar-)
  - [Operador Numeración: $](#operador-numeración-)
  - [Operador Subir de nivel: ^](#operador-subir-de-nivel-)
  - [Operador Agrupar: ()](#operador-agrupar-)
  - [Añadir Texto: {}](#añadir-texto-)
  - [Añadir Atributos: \[\]](#añadir-atributos-)
  - [- Añadir Clase: .](#--añadir-clase-)
  - [- Añadir ID: #](#--añadir-id-)
- [Comprimir lo Comprimido:](#comprimir-lo-comprimido)
- [Texto de Relleno](#texto-de-relleno)
- [Encapsular código](#encapsular-código)

## Sintaxis

La sintaxis principal de emmet son las etiquetas html sin los símbolos de apertura y cierre.

```HTML
<!-- Algunos Ejemplos: -->

Emmet: div

Resultado:
<div></div>

Emmet: a

Resultado:
<a href=””></a>

Emmet: img

Resultado:
<img src="" alt="">
```

## Operadores principales:

- **Hijo: >**
- **Hermano: +**
- **Multiplicar: \***
- **Numeración: $**
- **Subir Nivel: ^**
- **Agrupar: ()**
- **Añadir Texto: {}**
- **Añadir Atributo: []**
- **Añadir clase: .**
- **Añadir ID: #**

### Operador Hijo: >

> Usar el operador > para anidar elementos uno dentro del otro.

```HTML

Emmet: div>h1

Resultado:
<div>
   <h1></h1>
</div>

Emmet: div>article>p

Resultado:
<div>
    <article>
        <p></p>
    </article>
</div>
```

### Operador Hermano: +

> Use el operador + para colocar elementos en el mismo nivel.

```HTML

Emmet: div+p

Resultado:
<div></div>
<p></p>

Emmet: div+h1+h2

Resultado:
<div></div>
<h1></h1>
<h2></h2>
```

### Operador Multiplicar: \*

> Use el operador \* para definir el número de veces que se imprime el elemento.

```HTML

Emmet: div*3

Resultado:
<div></div>
<div></div>
<div></div>
```

### Operador Numeración: $

> Use el operador \$ para enumerar los elementos.
> Usando múltiples \$ se añaden ceros(antes del número entero).

```HTML

Emmet: ul>li*3{$}

Resultado:
<ul>
    <li>1</li>
    <li>2</li>
    <li>3</li>
</ul>

Emmet: ul>li*3{$$$$}

Resultado:
<ul>
    <li>0001</li>
    <li>0002</li>
    <li>0003</li>
</ul>
```

> Invertir el orden del operador numérico :

```HTML

Emmet: ul>li.item$@-*3

Resultado:
<ul>
    <li class="item3"></li>
    <li class="item2"></li>
    <li class="item1"></li>
</ul>

```

> Que comience con un número específico:

```HTML

Emmet: ul>li.item$@4*3

Resultado:
<ul>
    <li class="item4"></li>
    <li class="item5"></li>
    <li class="item6"></li>
</ul>


```

### Operador Subir de nivel: ^

> Use el operador ^ para subir de nivel los elementos.
>
> El operador hijo > añade los elementos en un "sub nivel".
>
> El operador hermano + añade los elementos en el mismo nivel.

```HTML

Emmet: div>ul>li^div>h1

Resultado:
<div>
    <ul>
        <li></li> <-Nivel antes de ^
    </ul>
    <div>  <-Nivel después de ^
        <h1></h1>
    </div>
</div>
```

### Operador Agrupar: ()

> Use el operador () para agrupar las etiquetas a la hora de estructurar una abreviatura compleja.
>
> Es otra forma de manejar los niveles de los elementos.

```HTML

Emmet: div>(header>ul>li>a)+footer>p

Resultado:
<div>
    <header>
        <ul>
            <li><a href=""></a></li>
        </ul>
    </header>
    <footer>
        <p></p>
    </footer>
</div>

```

### Añadir Texto: {}

> Use el operador {} para añadir texto dentro de los elementos.

```HTML

Emmet: div>h1{Encabezado}

Resultado:
<div>
    <h1>Encabezado</h1>
</div>
```

### Añadir Atributos: []

> Use el operador [] para añadir atributos a los elementos que se van a generar.

```HTML

Emmet: h1[style="color:red;"]{Encabezado en rojo}

Resultado:
<h1 style="color:red;">Encabezado en rojo</h1>
```

### - Añadir Clase: .

> Use el operador . para añadir clases a los elementos que se van a generar.

```HTML

Emmet: p.parrafo__principal{texto de relleno}

Resultado:
<p class="parrafo__principal">texto de relleno</p>

<!-- Usar Dos Clases. No Permite espacios(Evitar num en las clases) -->
Emmet: p.clase1.clase2

Resultado:
<p class="clase1 clase2"></p>
```

### - Añadir ID: \#

> Use el operador # para añadir un ID a los elementos que se van a generar.

```HTML

Emmet: p#parrafo__principal{texto de relleno}

Resultado:
<p id="parrafo__principal">texto de relleno</p>
```

## Comprimir lo Comprimido:

> No todos los comandos de emmet son compatibles con el operador **etiqueta:tipo/característica/atributo**.

-Ejemplos:

```HTML

Emmet: html:5 (Generar anatomía de un documento HTML 5)

Resultado:
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>

</body>
</html>

Emmet: form:get (Generar un formulario de tipo get)

Resultado:
<form action="" method="get"></form>

Emmet: a:mail (Generar un enlace de tipo mail)

Resultado:
<a href="mailto:"></a>

Emmet: input:submit (Generar un input de tipo submit)

Resultado:
<input type="submit" value="">
```

## Texto de Relleno

> Generar texto de relleno (Lorem ipsum):

```HTML

Emmet: lorem

Resultado:

Lorem ipsum dolor sit amet consectetur, adipisicing elit. Quae deleniti
voluptate, atque eligendi error, necessitatibus alias ipsam dolores
exercitationem quisquam facilis quo. Laudantium dolorum, perferendis vero
illum dolor facilis unde!
```

## Encapsular código

1- Seleccionar bloque de código.

2- Pulsar F1

3- Buscar Emmet:Wrap (Emmet:Encapsular)

4- Colocar comando de Emmet que encapsulara el código.








