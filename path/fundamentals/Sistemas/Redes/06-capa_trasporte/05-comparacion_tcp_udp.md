# Comparacion TCP vs UDP

TCP y UDP representan dos filosofias opuestas de comunicacion en la capa de transporte. TCP prioriza la fiabilidad y la integridad de los datos; UDP prioriza la velocidad y la baja latencia. La eleccion entre ambos tiene un impacto directo en el rendimiento y la experiencia de usuario de cualquier aplicacion de red.

No hay un protocolo "mejor" que otro: cada uno esta disenado para escenarios diferentes. Un servidor web necesita TCP para garantizar que cada pagina se carga completa; un juego en linea necesita UDP para enviar posiciones de jugadores en tiempo real sin esperar confirmaciones.

## Tabla comparativa

| Caracteristica | TCP | UDP |
|---|---|---|
| Conexion | Orientado a conexion (handshake) | Sin conexion |
| Fiabilidad | Confiable (ACK, retransmision) | No confiable (best effort) |
| Orden | Garantiza orden correcto | No garantiza orden |
| Control de flujo | Si (ventana deslizante) | No |
| Control de congestion | Si (slow start, congestion avoidance) | No |
| Cabecera | 20-60 bytes | 8 bytes |
| Latencia | Mayor (handshake + ACK) | Menor (sin overhead) |
| Overhead de red | Alto | Bajo |
| Consumo de CPU | Medio-Alto | Bajo |
| Velocidad efectiva | Limitada por control de flujo/congestion | Limitada solo por la red |

## Cuandos elegir TCP

TCP es la eleccion correcta cuando:

- **La integridad de datos es critica:** transferencia de archivos, correo electronico, bases de datos, paginas web con contenido dinamico.
- **El orden de los datos importa:** streaming de archivos completos (no en tiempo real), descargas, sincronizacion.
- **La aplicacion no puede perder datos:** transacciones bancarias, actualizaciones de software, mensajes de texto.

Ejemplos: HTTP/HTTPS, FTP, SMTP, SSH, Telnet, SCP, MySQL, PostgreSQL, SNMP.

## Cuandos elegir UDP

UDP es la eleccion correcta cuando:

- **La latencia es mas importante que la fiabilidad:** VoIP, videoconferencias, juegos en linea.
- **La perdida de algunos datos es tolerable:** streaming de video en tiempo real (un frame perdido causa un parpadeo, no corrompe el video).
- **La aplicacion implementa su propia fiabilidad:** QUIC (HTTP/3), WebRTC, protocolos de juegos.
- **Las operaciones son cortas y simples:** DNS, DHCP, NTP, SNMP (consultas simples).

Ejemplos: DNS, DHCP, VoIP (RTP), videoconferencia (WebRTC), juegos UDP, NTP, TFTP.

## TCP en la practica: overhead y latencia

El handshake de 3 vias de TCP introduce un minimo de un RTT (*Round-Trip Time*) antes de que se puedan enviar datos. En una conexion a un servidor a 100 ms de distancia, esto significa 100 ms adicionales antes del primer byte de datos. UDP no tiene esta penalizacion.

```
TCP:  SYN ──→  ←── SYN-ACK  ──→ ACK + datos   (1 RTT minimo)
UDP:  datos ──→                                     (0 RTT)
```

Esta diferencia es insignificante para transferencias de archivos largas pero critica para aplicaciones interactivas donde cada milisegundo cuenta.

## Protocolos hibridos: QUIC

QUIC (el protocolo base de HTTP/3) es un ejemplo de como UDP se utiliza como base para construir un protocolo con fiabilidad de TCP pero con menor latencia. QUIC implementa confirmaciones, retransmisiones y control de congestion a nivel de aplicacion sobre UDP, eliminando el handshake de 3 vias de TCP y permitiendo la transmision multiplexada sin head-of-line blocking.

```
TCP + TLS 1.3:  SYN → SYN-ACK → ACK → ClientHello → ServerHello → datos
                (3 RTT antes del primer byte de datos)

QUIC:           Initial → Handshake → datos
                (1 RTT antes del primer byte de datos)
```

> La eleccion entre TCP y UDP no es binaria en el mundo moderno. Protocolos como QUIC demuestran que es posible combinar la fiabilidad de TCP con la eficiencia de UDP, superando las limitaciones historicas de ambos. Comprender las fortalezas y debilidades de cada protocolo es esencial para disenar aplicaciones de red eficientes y para diagnosticar problemas de rendimiento.

