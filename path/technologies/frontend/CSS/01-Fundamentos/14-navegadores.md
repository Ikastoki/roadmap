<h1 align="center">Navegadores</h1>

<h2>📑 Contenido</h2>

- [¿Qué es la compatibilidad entre navegadores ?](#qué-es-la-compatibilidad-entre-navegadores-)
- [Sugerencias para garantizar una mayor compatibilidad](#sugerencias-para-garantizar-una-mayor-compatibilidad)
- [Resets](#resets)
  - [Ejemplo reset(simple)](#ejemplo-resetsimple)
  - [Reset de terceros](#reset-de-terceros)

## ¿Qué es la compatibilidad entre navegadores ?

La compatibilidad de CSS entre navegadores es un aspecto importante del desarrollo web, ya que diferentes navegadores pueden interpretar las reglas de estilo CSS de manera ligeramente diferente o admitir características específicas de CSS de manera diversa.

La compatibilidad entre navegadores es un desafío constante en el desarrollo web, pero con un enfoque cuidadoso y las mejores prácticas, puedes crear sitios web que funcionen de manera consistente en una variedad de navegadores.

## Sugerencias para garantizar una mayor compatibilidad

- **Usa estándares CSS:** Siempre es preferible seguir los estándares CSS establecidos por el W3C (Consorcio World Wide Web) en lugar de depender de propiedades o valores no estándar. Esto ayudará a garantizar una mayor compatibilidad entre navegadores.
- **Prefijos de proveedores:** Si necesitas usar propiedades CSS que son experimentales o específicas de un navegador, considera usar prefijos de proveedores como:
  - -webkit- (para WebKit)
  - -moz- (para Firefox),
  - -ms- (para Internet Explorer/Edge),
  - -o- (para Opera)
    > [!NOTE]
    >
    > Ten en cuenta que los navegadores modernos tienden a ser más compatibles con las versiones no prefijadas de estas propiedades.
- **Pruebas y ajustes:** Realiza pruebas exhaustivas de tu sitio web en diferentes navegadores y versiones para identificar problemas de compatibilidad. Utiliza herramientas de desarrollo y recursos en línea, como Can I use (https://caniuse.com/), para verificar la compatibilidad de propiedades y características específicas en diferentes navegadores.
- **Versiones de navegador obsoletas:** Ten en cuenta que las versiones más antiguas de los navegadores pueden no admitir ciertas características de CSS o pueden tener problemas de representación. Si es posible, considera establecer políticas de soporte mínimo de navegadores para tu proyecto.
- **Actualiza y mantén tu código:** Mantén tu código CSS actualizado y revisa regularmente para garantizar que siga siendo compatible con las últimas versiones de los navegadores. A medida que los navegadores se actualizan, es posible que ciertas incompatibilidades se resuelvan por sí solas.
- **Usa polyfills:** Considera el uso de polyfills, son scripts(JavaScript) que agregan funcionalidad a navegadores que no admiten ciertas características de CSS o JavaScript. Los polyfills pueden ayudar a mejorar la compatibilidad en navegadores antiguos.

## Resets

En CSS, los "resets" (también conocidos como "CSS resets" o "reset styles") son conjuntos de reglas CSS diseñadas para eliminar o restablecer los estilos predeterminados de los elementos HTML en los navegadores. La idea detrás de un reset de CSS es crear una base uniforme y consistente para el diseño web, independientemente de las diferencias en la forma en que los navegadores aplican estilos predeterminados a elementos HTML.

Los navegadores aplican estilos predeterminados a elementos HTML, como márgenes, rellenos, tamaños de fuente, colores de fondo, etc. Estos estilos predeterminados pueden variar entre navegadores y versiones, lo que a veces puede llevar a problemas de compatibilidad y dificultades para lograr la apariencia deseada en un sitio web.

Al utilizar un reset de CSS, se eliminan estos estilos predeterminados para que el diseñador o desarrollador tenga un mayor control sobre cómo se verán los elementos en la página. Un reset típico incluye reglas como establecer los márgenes y rellenos en 0, establecer el tamaño de fuente en un valor consistente, y eliminar cualquier decoración de texto subrayado o estilo de viñeta de las listas, entre otros.

### Ejemplo reset(simple)

```css
/* Reset CSS */
html,
body,
div,
span,
h1,
h2,
h3,
p,
a,
img,
ul,
li {
  margin: 0;
  padding: 0;
  border: 0;
  font-size: 100%;
  font: inherit;
  vertical-align: baseline;
}

/* Evitar decoración de texto subrayado en enlaces */
a {
  text-decoration: none;
}
```

### Reset de terceros

El [Reset de Meyer](https://meyerweb.com/eric/tools/css/reset/), Eric A.Meyer fue uno de los primeros en compartir una hoja de restablecimiento para CSS. Como indican en su propia web, estos estilos de reinicio son muy genéricos. Por este motivo recomiendan que modifiques el archivo ajustándolo a tu proyecto, que lo uses como un punto de partida.

El [Reset de Richard Clark](http://html5doctor.com/html-5-reset-stylesheet/), Richard Clark modifico el reset de Meyer adaptándolo a HTML5. Si observas el código veras que añade elemento y atributos de selector que no se encuentran en el de Meyer.

[Normalize](https://necolas.github.io/normalize.css/) **no es un CSS reset como tal**, no restablece todos los estilos por defecto pero si hace que sea mas consistente en múltiples navegadores. Su objetivo es igualar los motores de renderizado de los diferentes navegadores.

> [!NOTE]
>
> Es importante tener en cuenta que los resets de CSS no son necesarios en todos los proyectos y pueden ser excesivos en algunos casos. A menudo, se prefieren en proyectos donde se busca un alto nivel de personalización del diseño y se desea comenzar desde una base limpia y uniforme.


