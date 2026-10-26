# El modelo OSI de 7 capas

El modelo OSI (*Open Systems Interconnection*) es un marco de referencia teorico de la ISO que describe las siete capas necesarias para la comunicacion entre sistemas abiertos. Cada capa tiene una funcion especifica y se comunica con las capas adyacentes a traves de interfaces bien definidas. OSI es la herramienta educativa fundamental para comprender las redes, aunque en la practica se utilizan los protocolos TCP/IP.

Las siete capas, desde la mas cercana al hardware hasta la mas cercana al usuario, son: Fisica, Enlace de datos, Red, Transporte, Sesion, Presentacion y Aplicacion. Cada una resuelve un problema concreto de la comunicacion.

## Capa 1: Fisica

La capa fisica es responsable de la transmision y recepcion de bits brutos sobre un medio fisico. Define las caracteristicas electricas, mecanicas y temporales de la interfaz: voltajes, tiempos de uso, velocidades de transmision, tipos de cable y conectores.

Esta capa no interpreta los bits: solo los envia y recibe como una secuencia de 0 y 1. No tiene notion de paquetes, direcciones ni errores. Su unica mision es transportar bits de forma fiable a traves del medio fisico.

Ejemplos: cable Ethernet Cat6, fibra optica, especificaciones electricas RS-232, modulacion WiFi.

## Capa 2: Enlace de datos

La capa de enlace de datos organiza los bits en **tramas** (*frames*) y proporciona transferencia de datos confiable entre dos nodos directamente conectados en la misma red local. Detecta y corrige errores que ocurran en la capa fisica y controla el acceso al medio para evitar colisiones.

Esta capa se subdivide en dos subcapas: MAC (*Media Access Control*), que controla el acceso fisico al medio y las direcciones de hardware; y LLC (*Logical Link Control*), que multiplexa protocolos y proporciona control de errores basico.

Ejemplos: Ethernet (802.3), Wi-Fi (802.11), PPP, frames HDLC.

## Capa 3: Red

La capa de red es responsable del enrutamiento y el direccionamiento logico. Determina el camino que deben seguir los datos a traves de multiples redes interconectadas, desde la fuente hasta el destino, usando direcciones IP como identificadores unicos.

Esta capa toma decisiones de enrutamiento: que router转发 los paquetes, que ruta es mas corta o mas eficiente. Tambien gestiona la fragmentacion y reensamblado de paquetes cuando la MTU (*Maximum Transmission Unit*) del medio es menor que el tamano del paquete.

Ejemplos: IPv4, IPv6, ICMP, ARP, protocolos de enrutamiento (OSPF, BGP).

## Capa 4: Transporte

La capa de transporte proporciona transferencia de datos de extremo a extremo entre procesos en host origen y host destino. A diferencia de la capa de red (que direccione hosts), la capa de transporte direccione procesos individuales usando puertos como identificadores.

Ofrece dos niveles de servicio: TCP, orientado a conexion y confiable; y UDP, sin conexion y sin garantias de entrega. Tambien gestiona el control de flujo y la multiplexacion de conexiones.

Ejemplos: TCP, UDP, SCTP.

## Capa 5: Sesion

La capa de sesion gestiona el establecimiento, mantenimiento y terminacion de sesiones de comunicacion entre aplicaciones. Proporciona mecanismos de sincronizacion y control de dialogo, permitiendo que las aplicaciones mantengan conversaciones estructuradas.

En la practica, las funciones de la capa de sesion suelen ser asumidas por las capas inferiores (TCP para el control de conexion) o por las propias aplicaciones. Por esta razon, TCP/IP no tiene una capa de sesion explicita.

## Capa 6: Presentacion

La capa de presentacion se encarga de la representacion, compresion y cifrado de los datos. Asegura que la informacion enviada por la capa de aplicacion de un sistema sea legible por la capa de aplicacion del sistema receptor.

Traduce entre formatos de datos (por ejemplo, EBCDIC a ASCII), gestiona la compresion de datos y proporciona servicios de cifrado. Como la capa de sesion, en TCP/IP estas funciones suelen ser asumidas por la aplicacion.

## Capa 7: Aplicacion

La capa de aplicacion es la interfaz entre la red y las aplicaciones del usuario. Proporciona servicios de red a las aplicaciones: transferencia de archivos, correo electronico, acceso a web, resolucion de nombres de dominio, etc.

Esta capa no se refiere a las aplicaciones en si, sino a los protocolos que utilizan para comunicarse a traves de la red. HTTP, SMTP, FTP, SSH y DNS son protocolos de la capa de aplicacion.

## Resumen

| Capa | Nombre | Funcion principal | PDU |
|---|---|---|---|
| 7 | Aplicacion | Servicios a aplicaciones | Datos |
| 6 | Presentacion | Formato, cifrado, compresion | Datos |
| 5 | Sesion | Sincronizacion de sesiones | Datos |
| 4 | Transporte | Extremo a extremo, puertos | Segmento |
| 3 | Red | Enrutamiento, IP | Paquete |
| 2 | Enlace de datos | Tramas, MAC, deteccion de errores | Trama |
| 1 | Fisica | Bits en el medio | Bits |

> El modelo OSI es una referencia conceptual, no una implementacion. En la practica, las capas 5, 6 y 7 rara vez se implementan como capas separadas en TCP/IP. Sin embargo, la terminologia OSI sigue siendo la base para describir y diagnosticar problemas de red: cuando un ingeniero dice "problema de capa 3", todos entienden que se refiere a un problema de enrutamiento o direccionamiento IP.

