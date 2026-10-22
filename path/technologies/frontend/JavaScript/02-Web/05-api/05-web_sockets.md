<h1 align="center"></h1>

<h2>📑 Contenido</h2>

- [Web Sockets](#web-sockets)
- [¿Cómo funcionan los Web Sockets?](#cómo-funcionan-los-web-sockets)
- [Ventajas de los Web Sockets](#ventajas-de-los-web-sockets)
- [Ejemplo de uso](#ejemplo-de-uso)
- [Web Sockets vs HTTP](#web-sockets-vs-http)

## Web Sockets

Los Web Sockets son una tecnología de comunicación web que permite un canal de comunicación bidireccional y persistente entre un cliente (generalmente un navegador) y un servidor. A diferencia de las peticiones tradicionales HTTP, que son de una sola vía (el cliente pide, el servidor responde y la conexión se cierra), los Web Sockets mantienen la conexión abierta, lo que permite el envío de datos en tiempo real.

## ¿Cómo funcionan los Web Sockets?

El proceso de un Web Socket se puede dividir en dos fases:

- Handshake (Apertura): La comunicación comienza con una petición HTTP normal del cliente al servidor. En esta petición, el cliente solicita un "cambio de protocolo" (un `Upgrade`) a Web Sockets. Si el servidor lo acepta, responde con un código de estado `101 Switching Protocols`, y la conexión se transforma en una conexión Web Socket persistente.

- Transmisión de datos: Una vez que la conexión está abierta, el cliente y el servidor pueden enviarse mensajes mutuamente en cualquier momento sin necesidad de una nueva petición. Esto es lo que permite la comunicación en tiempo real.

## Ventajas de los Web Sockets

- Comunicación en tiempo real: Es la principal ventaja. Permite el desarrollo de aplicaciones que requieren actualizaciones instantáneas, como chats en vivo, juegos multijugador, notificaciones push, y dashboards en tiempo real.

- Eficiencia: Como la conexión está siempre abierta, se eliminan los costos de establecer y cerrar una conexión HTTP en cada intercambio de datos. Esto reduce la latencia y la carga en el servidor.

- Comunicación bidireccional: Tanto el cliente como el servidor pueden iniciar la comunicación en cualquier momento.

## Ejemplo de uso

El uso de Web Sockets es relativamente sencillo. El navegador proporciona el objeto global WebSocket.

En el lado del cliente (JavaScript):

```javascript
// Crear una nueva instancia de WebSocket
const ws = new WebSocket("wss://api.ejemplo.com/chat");

// Escuchar eventos de la conexión
ws.onopen = (event) => {
  console.log("Conexión establecida.");
  ws.send("Hola, soy el cliente."); // Enviar un mensaje al servidor
};

ws.onmessage = (event) => {
  console.log("Mensaje del servidor:", event.data); // Recibir un mensaje del servidor
};

ws.onclose = (event) => {
  console.log("Conexión cerrada.");
};

ws.onerror = (event) => {
  console.error("Error en WebSocket:", event);
};
```

En el lado del servidor, necesitarías un backend que soporte Web Sockets, como Node.js con la librería ws o un framework como Express con Socket.io.

## Web Sockets vs HTTP

| Característica | Peticiones HTTP                      | Web Sockets                      |
| -------------- | ------------------------------------ | -------------------------------- |
| Modelo         | Cliente pide, servidor responde      | Comunicación bidireccional       |
| Conexión       | No persistente, se abre y se cierra  | Persistente, se mantiene abierta |
| Latencia       | Alta, por el costo de abrir y cerrar | Baja, conexión instantánea       |
| Uso            | Carga de páginas, APIs REST          | Chats, notificaciones, juegos    |
