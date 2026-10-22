<h1 align="center">Node JS</h1>

<h2>📑 Contenido</h2>

- [Node JS](#node-js)
- [Características clave:](#características-clave)
- [¿Para qué se usa Node.js?](#para-qué-se-usa-nodejs)
- [Diferencias clave con JavaScript en el navegador](#diferencias-clave-con-javascript-en-el-navegador)

## Node JS

Node.js es un entorno de ejecución para JavaScript construido sobre el motor V8 de Google Chrome. En pocas palabras, Node.js te permite ejecutar código JavaScript fuera de un navegador web, típicamente en un servidor, en tu computadora de escritorio o en otros dispositivos.

Antes de Node.js, JavaScript se limitaba al cliente (el navegador) para hacer que las páginas web fueran interactivas. Con Node.js, JavaScript se convirtió en un lenguaje versátil para el desarrollo del lado del servidor.

## Características clave:

- Basado en V8: Node.js utiliza el mismo motor de JavaScript ultra-rápido de Google Chrome, lo que le da un excelente rendimiento.

- Modelo de E/S no bloqueante y asíncrono: Esta es la característica más importante de Node.js. En lugar de esperar a que una tarea se complete (como leer un archivo o hacer una petición a una base de datos), Node.js la delega y continúa ejecutando otras tareas. Cuando la operación asíncrona termina, Node.js recibe una notificación y la maneja. Esto lo hace muy eficiente para manejar muchas conexiones simultáneas, como en un servidor web.

- Un solo hilo (Single-threaded): A diferencia de otros entornos de servidor que crean un nuevo hilo para cada conexión, Node.js funciona con un solo hilo principal. Esto se complementa con su modelo asíncrono, permitiéndole manejar la concurrencia de forma eficiente sin la sobrecarga de los hilos.

- Ecosistema de NPM: Node.js viene con NPM (Node Package Manager), el gestor de paquetes más grande del mundo, que te da acceso a una vasta colección de bibliotecas y herramientas para acelerar el desarrollo.

## ¿Para qué se usa Node.js?

Node.js es ideal para aplicaciones que necesitan manejar muchas operaciones de entrada/salida (I/O) en tiempo real, como:

- Servidores web y APIs REST: Es la principal razón de su popularidad. Frameworks como Express.js simplifican enormemente la creación de servidores.

- Aplicaciones en tiempo real: Chats, juegos multijugador, etc., que se benefician de la comunicación instantánea de Web Sockets.

- Herramientas de línea de comandos (CLI): Muchas herramientas de desarrollo front-end, como Webpack, son construidas con Node.js.

## Diferencias clave con JavaScript en el navegador

| Característica | JavaScript en el Navegador                    | Node.js                                                  |
| -------------- | --------------------------------------------- | -------------------------------------------------------- |
| Entorno        | El navegador (Chrome, Firefox, etc.)          | El entorno de ejecución de Node.js                       |
| APIs           | DOM, window, localStorage, fetch              | Módulos nativos (fs para ficheros, http para servidores) |
| Modelo         | Síncrono, con APIs asíncronas como setTimeout | Asíncrono y no bloqueante por defecto                    |
