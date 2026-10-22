<h1 align='center'>Entornos de Ejecución</h1>

<h2>📑 Contenido</h2>

- [Entornos de Ejecución](#entornos-de-ejecución)
- [¿Qué es un entorno de ejecución?](#qué-es-un-entorno-de-ejecución)
- [Tipos de entornos de ejecución en JavaScript](#tipos-de-entornos-de-ejecución-en-javascript)
- [Entorno del Navegador](#entorno-del-navegador)
  - [Pequeñas diferencias](#pequeñas-diferencias)
  - [Cómo manejar las diferencias entre navegadores](#cómo-manejar-las-diferencias-entre-navegadores)
- [Entorno de Servidor (Node.js)](#entorno-de-servidor-nodejs)

## Entornos de Ejecución

En este capítulo exploraremos qué es un entorno de ejecución en JavaScript, sus tipos y cómo influyen en el funcionamiento del código. Los entornos de ejecución son fundamentales para comprender cómo y dónde se ejecuta el código JavaScript, ya sea en un navegador o en un servidor, y cómo interactúa con los recursos disponibles.

## ¿Qué es un entorno de ejecución?

El entorno de ejecución (runtime environment) es el contexto en el cual se ejecuta el código de JavaScript. En palabras sencillas, es el "lugar" donde el código se interpreta y lleva a cabo. Cada entorno de ejecución ofrece un conjunto de herramientas, APIs y funcionalidades específicas, que permiten al código interactuar con el sistema operativo, la red, el hardware u otros recursos.

Ejemplo para visualizarlo:

Imagina que el código JavaScript es como una receta para cocinar. El entorno de ejecución sería la cocina (el lugar donde cocinas), y esta cocina podría estar equipada de diferentes formas: con utensilios, ingredientes, y electrodomésticos. Dependiendo de qué "cocina" estés usando (navegador o servidor), tendrás acceso a diferentes herramientas.

## Tipos de entornos de ejecución en JavaScript

JavaScript puede ejecutarse en varios tipos de entornos, pero los dos principales son:

- Entorno del Navegador (Browser Environment)
- Entorno de Servidor (Node.js)

## Entorno del Navegador

El entorno de ejecución más común para JavaScript es el navegador web, como Google Chrome, Mozilla Firefox, Safari o Microsoft Edge. JavaScript fue creado inicialmente para ejecutarse en navegadores, con el objetivo de hacer que las páginas web sean interactivas.

En este entorno, JavaScript tiene acceso a varias APIs proporcionadas por el navegador, las cuales permiten manipular el contenido de la página y responder a los eventos del usuario. A continuación, algunos de los elementos clave en el entorno del navegador:

- **DOM (Document Object Model):** Una API que permite a JavaScript interactuar y modificar el contenido HTML y los estilos CSS de la página web. Cada elemento en la página web se representa como un nodo dentro de un árbol, y JavaScript puede acceder, modificar, eliminar o añadir nodos.

- **BOM (Browser Object Model):** Ofrece objetos que permiten interactuar con el navegador en sí, como window, navigator o history. Por ejemplo, con el objeto window puedes manipular el tamaño de la ventana del navegador o abrir nuevas pestañas.

- **Eventos:** JavaScript puede escuchar eventos del navegador o del usuario (como clics, desplazamientos, teclas presionadas), y reaccionar a ellos en tiempo real.

> [!NOTE]
> Una API (Interfaz de Programación de Aplicaciones, por sus siglas en inglés: Application Programming Interface) es un conjunto de reglas y definiciones que permite que diferentes programas o sistemas se comuniquen entre sí. Las APIs definen las formas en que los desarrolladores pueden interactuar con una aplicación, servicio o base de datos, sin necesidad de saber cómo funciona internamente.

### Pequeñas diferencias

JavaScript no se ejecuta exactamente igual en todos los navegadores, aunque las diferencias no son tan grandes ni evidentes como en CSS. Todos los navegadores modernos siguen el anteriormente mencionado estándar ECMAScript, que define cómo debe funcionar JavaScript, pero hay algunas diferencias sutiles en la implementación que pueden afectar el comportamiento del código JavaScript en diferentes navegadores.

Cada navegador utiliza un motor de JavaScript distinto:

- Google Chrome y Microsoft Edge: Usan el motor V8.
- Mozilla Firefox: Usa SpiderMonkey.
- Safari: Usa JavaScriptCore.

Aunque estos motores siguen el estándar ECMAScript, cada uno tiene su propia implementación y optimizaciones, lo que puede generar pequeñas diferencias en cómo se ejecuta el código JavaScript. A veces, la velocidad o el comportamiento de ciertas operaciones puede variar ligeramente entre los motores. Los navegadores no siempre implementan las nuevas características de ECMAScript al mismo tiempo.

> [!TIP]
> Para verificar qué características de ECMAScript son compatibles con los navegadores, los desarrolladores usan herramientas como Can I Use.

### Cómo manejar las diferencias entre navegadores

Si quieres usar una característica moderna que no está soportada en todos los navegadores, puedes utilizar un polyfill. Los polyfills son fragmentos de código que imitan las nuevas funcionalidades en navegadores más antiguos. También herramientas como Babel te permiten escribir código JavaScript moderno y luego "transpilarlo" (convertirlo) a una versión más antigua y compatible con todos los navegadores. Esto es útil si estás utilizando características de ECMAScript que no están ampliamente soportadas aún. Babel se encarga de convertir características como funciones flecha, clases, y async/await en versiones más antiguas del lenguaje.

## Entorno de Servidor (Node.js)

Node.js es un entorno de ejecución para JavaScript en el servidor. Fue lanzado en 2009 y cambió el panorama del desarrollo web al permitir que los desarrolladores usen JavaScript tanto en el lado del cliente (navegador) como en el lado del servidor.

A diferencia del entorno del navegador, en Node.js no tienes acceso directo al DOM o a APIs propias del navegador, ya que no estás interactuando con una página web. Sin embargo, Node.js proporciona acceso a:

- **Sistema de archivos (File System):** Con la API fs puedes leer y escribir archivos en el disco.

- **Redes:** Node.js permite la creación de servidores web y la gestión de solicitudes HTTP, permitiendo construir aplicaciones back-end o API con JavaScript.

- **Módulos:** Node.js tiene un sistema de módulos integrado que facilita la reutilización y organización del código. También permite importar bibliotecas externas mediante npm (Node Package Manager).

> [!WARNING]
> Esto solo es una breve introducción, la parte Web de JavaScript y NodeJs son dos bloques muy amplios que necesitan su propio apartado.
