<h1 align="center"> Markdown</h1>

<h2>Contenido:</h2>

- [Introducción](#introducción)
- [Encabezados](#encabezados)
- [Párrafos](#párrafos)
- [Cursiva y Negrita](#cursiva-y-negrita)
- [Enlaces](#enlaces)
- [Imágenes](#imágenes)
- [Listas](#listas)
- [Remarcar](#remarcar)
- [Código](#código)
- [Tablas](#tablas)
- [HTML](#html)

## Introducción

Markdown es un lenguaje de marcado ligero diseñado para ser fácil de leer y escribir, mientras permite la conversión sencilla a HTML y otros formatos de presentación. Fue creado por John Gruber y Aaron Swartz en 2004 y se ha convertido en una forma popular de formatear texto para la web, especialmente en blogs, foros y plataformas de colaboración.

Markdown se puede utilizar en una amplia variedad de aplicaciones, desde documentos simples hasta publicaciones en blogs, correos electrónicos, wikis y más. Puedes escribir en Markdown directamente en muchos editores de texto y plataformas, y luego convertirlo a HTML o a otros formatos de salida según sea necesario. Puedes utilizar [markdown to html](https://markdowntohtml.com/) para renderizarlo como HTML.

## Encabezados

```Markdown
<h1 style="text-decoration: underline; color: #6A67CE"> Encabezados "CSS"</h1>

# Encabezado 1

## Encabezado 2

### Encabezado 3

#### Encabezado 4

##### Encabezado 5

###### Encabezado 6
```

## Párrafos

Párrafo de prueba en markdown.

## Cursiva y Negrita

```Markdown
Frase normal
_Frase en cursiva_
**Frase en negrita**
```

## Enlaces

```Markdown
Visita este enlace [Markdown](https://es.wikipedia.org/wiki/Markdown) para saber más.

[Subir(enlace interno)](#encabezados)
```

## Imágenes

```Markdown
![Lorem Picsum](https://picsum.photos/300)
```

## Listas

```Markdown
- Elemento 1
- Elemento 2
- Elemento 3

1. Elemento 1
2. Elemento 2
3. Elemento 3
```

## Remarcar

```Markdown
> Texto Remarcado en markdown
```

## Código

```bash
//Bash
ping 8.8.8.8
```

```javascript
//JavaScript
const saludo = "Hola";
console.log(saludo);
```

## Tablas

```Markdown
| Id  | Nombre | Tel       | E-Mail         |
| :-: | ------ | --------- | -------------- |
|  1  | User 1 | 123456789 | User1@mail.com |
|  2  | User 2 | 987654321 | User2@mail.com |
```

| Id  | Nombre | Tel       | E-Mail         |
| :-: | ------ | --------- | -------------- |
|  1  | User 1 | 123456789 | User1@mail.com |
|  2  | User 2 | 987654321 | User2@mail.com |

## HTML

Markdown y HTML son dos lenguajes diferentes utilizados para dar formato a documentos web, y generalmente no se mezclan directamente en el mismo documento. Sin embargo, algunos sistemas que admiten Markdown permiten la inclusión de código HTML para casos en los que se necesite más control sobre el formato.

> [!TIP]
>
> Utilizar HTML en Markdown puede ser aconsejable en situaciones específicas donde necesitas funcionalidades o estilos que no están cubiertos por las capacidades estándar de Markdown.
>
> - Si necesitas incorporar formularios interactivos, elementos multimedia, o contenido incrustado que no está cubierto por la sintaxis estándar de Markdown.
> - Si quieres aplicar estilos y diseños personalizados que no pueden lograrse fácilmente con la sintaxis de Markdown.
> - Compatibilidad con otras tecnologías, gestionar contenido CMS.

En general, puedes incluir código HTML directamente en un documento Markdown y, en muchos casos, se interpretará correctamente.

> [!WARNING]
>
> Ten en cuenta que esta práctica de mezclar Markdown con HTML no es estándar y puede no funcionar en todas las implementaciones de Markdown. Además, la capacidad de usar HTML dentro de Markdown a menudo depende de la configuración o las restricciones del sistema en el que estés trabajando.

> [!CAUTION]
> 
> El objetivo principal de Markdown es proporcionar una sintaxis simple y legible para la creación de contenido web. Al introducir HTML, puedes perder la portabilidad y la legibilidad que ofrece Markdown. Además, el uso excesivo de HTML puede hacer que el contenido sea más difícil de mantener y comprender, especialmente para aquellos que no estén familiarizados con HTML. En la medida de lo posible, es preferible aprovechar las capacidades nativas de Markdown.
