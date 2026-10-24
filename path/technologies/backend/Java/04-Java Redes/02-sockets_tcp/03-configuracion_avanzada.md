# Configuracion avanzada de sockets

Los sockets TCP en Java exponen multiples opciones de configuracion que permiten ajustar el comportamiento de la capa de transporte a las necesidades especificas de la aplicacion. Un conocimiento profundo de estas opciones es necesario para construir sistemas robustos y eficientes.

## Timeouts

Los timeouts evitan que una aplicacion quede bloqueada indefinidamente cuando la red falla o el extremo remoto no responde.

### setSoTimeout

Establece el tiempo maximo que `read()` esperara por datos. Si transcurre sin recibir nada, lanza `java.net.SocketTimeoutException`. El valor por defecto es 0 (espera infinita).

```java
Socket socket = new Socket("host", 8080);
socket.setSoTimeout(5000); // 5 segundos
```

En `ServerSocket`, `setSoTimeout()` afecta al metodo `accept()`: si no llega ninguna conexion en el tiempo especificado, se lanza la misma excepcion.

### Connect timeout

El constructor `Socket(String host, int port)` bloquea hasta que la conexion TCP se establece o falla, sin limite configurable. Para controlar el tiempo de conexion se usa un constructor alternativo o `Socket.connect()`.

```java
Socket socket = new Socket();
SocketAddress addr = new InetSocketAddress("host", 8080);
socket.connect(addr, 3000); // timeout de conexion de 3 segundos
```

Si el servidor no responde al SYN en 3 segundos, se lanza `java.net.ConnectException: Connection timed out`.

## Buffers de envio y recepcion

TCP mantiene buffers internos en el kernel del sistema operativo para almacenar datos antes de enviarlos por la red o despues de recibirlos. `setSendBufferSize()` y `setReceiveBufferSize()` permiten ajustar su tamano.

```java
Socket socket = new Socket();
socket.setSendBufferSize(65536);  // 64 KB
socket.setReceiveBufferSize(65536);
```

> Estos metodos sugieren un tamano al sistema operativo, que puede ignorarlo o ajustarlo a un maximo definido por el kernel (tipicamente entre 64 KB y 16 MB segun el sistema). La unica forma de confirmar el tamano asignado es leerlo con `getSendBufferSize()` y `getReceiveBufferSize()` despues de la conexion.

Un buffer de recepcion grande permite al kernel acumular mas datos antes de que la aplicacion los lea, mejorando el rendimiento en conexiones con alta latencia o productos ancho de banda (BDP - Bandwidth Delay Product). Un buffer pequeno reduce el uso de memoria pero puede provocar perdida de rendimiento si la aplicacion no lee con suficiente frecuencia.

## Opciones TCP nativas

Java expone varias opciones de la capa TCP mediante metodos de `Socket` y `ServerSocket`.

### TcpNoDelay

Por defecto, TCP aplica el algoritmo de Nagle, que acumula pequenos segmentos antes de enviarlos para mejorar la eficiencia de la red. `setTcpNoDelay(true)` desactiva Nagle y envia cada escritura inmediatamente como un segmento independiente.

```java
socket.setTcpNoDelay(true);
```

Es especialmente util en protocolos interactivos o de baja latencia donde los mensajes son pequenos y no deben retardarse. El coste es un mayor numero de segmentos en la red y posiblemente mas overhead.

### KeepAlive

`setKeepAlive(true)` activa el envio periodico de segmentos keep-alive TCP para detectar si el extremo remoto sigue accesible. Si no hay respuesta tras varios intentos (configurables en el sistema operativo, no en Java), la conexion se cierra.

```java
socket.setKeepAlive(true);
```

> KeepAlive no debe usarse como mecanismo de heartbeat para aplicaciones. Su periodo tipico es de 2 horas en Linux antes del primer probe, y los valores solo pueden cambiarse a nivel de sistema. Para detectar desconexiones rapidamente, es mejor implementar un heartbeat a nivel de aplicacion.

### ReuseAddress

Permite reutilizar una direccion y puerto que esta en estado TIME_WAIT. Sin esta opcion, al reiniciar un servidor puede ser necesario esperar varios minutos (tipicamente 2 * MSL) antes de poder vincular el mismo puerto.

```java
ServerSocket ss = new ServerSocket();
ss.setReuseAddress(true);
ss.bind(new InetSocketAddress(8080));
```

Es practicamente obligatorio en servidores que se reinician con frecuencia o que escalan horizontalmente con puertos compartidos.

### SoLinger

Controla el comportamiento del cierre de la conexion. Por defecto, `close()` retorna inmediatamente y TCP completa el envio de los datos pendientes en segundo plano. Con `setSoLinger(true, segundos)`, `close()` bloquea hasta que los datos pendientes se hayan enviado y reconocido, o hasta que transcurra el tiempo especificado.

```java
socket.setSoLinger(true, 10); // espera hasta 10 segundos
socket.setSoLinger(false, 0); // comportamiento por defecto
```

Envio un valor de 0 segundos, `close()` envia un RST en lugar de FIN, descartando cualquier dato pendiente. Es una opcion drastica que solo deberia usarse cuando se esta seguro de que no es necesario completar la transmision.

## Proxy

Java soporta conexiones a traves de proxies HTTP CONNECT y SOCKS mediante la clase `Proxy`.

### Proxy HTTP CONNECT

```java
SocketAddress proxyAddr = new InetSocketAddress("proxy.ejemplo.com", 3128);
Proxy proxy = new Proxy(Proxy.Type.HTTP, proxyAddr);
Socket socket = new Socket(proxy);
socket.connect(new InetSocketAddress("destino.com", 443));
```

### Proxy SOCKS

```java
SocketAddress proxyAddr = new InetSocketAddress("socks.ejemplo.com", 1080);
Proxy proxy = new Proxy(Proxy.Type.SOCKS, proxyAddr);
Socket socket = new Socket(proxy);
socket.connect(new InetSocketAddress("destino.com", 80));
```

Tambien es posible configurar proxies a nivel de sistema con las propiedades `socksProxyHost`, `socksProxyPort`, `http.proxyHost` y `http.proxyPort`, aunque el uso explicito de `Proxy` ofrece mayor control y claridad.

> Los proxies HTTP CONNECT funcionan estableciendo un tunel TCP: el cliente envia una peticion CONNECT al proxy, y este reenvia los bytes bidireccionalmente entre el cliente y el destino. SOCKS opera en la capa 5 del modelo OSI y soporta cualquier protocolo, no solo HTTP.
