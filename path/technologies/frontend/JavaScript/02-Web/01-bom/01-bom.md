<h1 align='center'>BOM - Browser Object Model</h1>

<h2>📑 Contenido</h2>

- [BOM - Browser Object Model](#bom---browser-object-model)
- [¿Qué es el BOM?](#qué-es-el-bom)
  - [Características clave del BOM:](#características-clave-del-bom)

## BOM - Browser Object Model

El Browser Object Model (BOM) es un conjunto de objetos que el navegador expone a JavaScript para interactuar con el entorno del navegador fuera de la página web. A través del BOM, JavaScript puede acceder y controlar aspectos como la ventana del navegador, el historial de navegación, la URL actual, la pantalla del dispositivo y otras funcionalidades específicas del navegador.

A diferencia del DOM (Document Object Model), que se ocupa de la estructura y contenido del documento HTML, el BOM se ocupa del navegador como plataforma.

## ¿Qué es el BOM?

El BOM representa todas las funcionalidades proporcionadas por el navegador para interactuar con su entorno. Mientras que el DOM se enfoca en la página web, el BOM cubre una serie de APIs que permiten controlar el navegador y su entorno. Algunos ejemplos de esto incluyen redireccionar a una nueva página, controlar la ventana del navegador, obtener información sobre la pantalla del dispositivo, y manejar el historial de navegación.

### Características clave del BOM:

- **Objeto window:** Es el objeto raíz del BOM y representa la ventana o marco del navegador. A través de window, se puede acceder a otros objetos como location, history, navigator, y screen.
- **Interacción con la URL:** Con el objeto location, el BOM permite manipular la URL actual, redirigir al usuario a otras páginas y recargar la página.
- **Gestión del historial:** El objeto history permite navegar hacia atrás y hacia adelante en el historial de navegación del usuario.
- **Control de ventanas:** El objeto window incluye métodos para abrir y cerrar ventanas emergentes o modificar el tamaño de la ventana.

El BOM no es parte del estándar ECMAScript, lo que significa que puede variar ligeramente entre diferentes navegadores.
