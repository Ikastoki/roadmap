<h1 align="center">Listas Anidadas y Enlaces</h1>

<h2>📑 Contenido</h2>

- [Listas Anidadas](#listas-anidadas)
- [Listas de Enlaces](#listas-de-enlaces)

## Listas Anidadas

Se pueden anidar los diferentes tipos de listas para formar una lista que contenga: listas ordenadas, desordenadas, de definición, con enlaces, imágenes...

```HTML
Ejemplo:
  <ul>
      <li>Elemento 1</li>
      <li>Elemento 2</li>
      <li>
        <ol>
          <li>Elemento 2.1</li>
          <li>Elemento 2.2</li>
        </ol>
      </li>
      <li>Elemento 3</li>
      <dd>
        <dt>Término1</dt>
        <dd>Definición 1</dd>
        <dt>Término 2</dt>
        <dd>Definición 2</dd>
      </dd>
      <li>Elemento 4</li>
    </ul>

Resultado:
    • Elemento 1
    • Elemento 2
        1. Elemento 2.1
        2. Elemento 2.2
    • Elemento 3
    Término1
        Definición 1
    Término 2
        Definición 2
    • Elemento 4
```

## Listas de Enlaces

Este tipo de listas son muy comunes a la hora de crear una navegación para una página web.

```HTML
Ejemplo:
     <ul>
      <li><a href="ruta_del_enlace">Enlace 1</a></li>
      <li><a href="ruta_del_enlace">Enlace 2</a></li>
      <li><a href="ruta_del_enlace">Enlace 3</a></li>
    </ul>
```

Resultado:

- <u>Enlace 1</u>
- <u>Enlace 2</u>
- <u>Enlace 3</u>


