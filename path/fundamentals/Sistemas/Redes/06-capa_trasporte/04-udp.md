# UDP (User Datagram Protocol)

UDP es el protocolo de transporte sin conexion y sin garantias de entrega. A diferencia de TCP, UDP no establece una conexion antes de enviar datos, no confirma la recepcion y no retransmite segmentos perdidos. Este minimalismo lo convierte en el protocolo ideal para aplicaciones donde la velocidad y la baja latencia son mas importantes que la fiabilidad.

UDP es preferido cuando la perdida ocasional de datos es aceptable o cuando la aplicacion implementa sus propios mecanismos de fiabilidad. El overhead minimo de UDP (solo 8 bytes de cabecera frente a los 20-60 bytes de TCP) permite una transmision mas rapida y con menor latencia.

## Caracteristicas de UDP

- **Sin conexion:** no hay handshake de 3 vias ni terminacion ordenada. El emisor simplemente envia datagramas sin establecer estado alguno.
- **Sin confirmacion (no confiable):** no hay ACK, no hay retransmisiones, no hay garantia de entrega.
- **Sin orden:** los datagramas pueden llegar fuera de orden; UDP no los reordena.
- **Sin control de congestion:** UDP envia datos a la velocidad que la aplicacion solicita, sin adaptarse a las condiciones de la red.
- **Cabecera minima:** 8 bytes (puertos origen/destino, longitud, checksum).

## Cabecera UDP

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Puerto Origen         │         Puerto Destino           │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│             Longitud           │           Checksum               │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                            Datos                                  │
└─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
```

La simplicidad de la cabecera UDP reduce el overhead de procesamiento en emisor y receptor, lo que se traduce en menor latencia.

## Casos de uso de UDP

**DNS (*Domain Name System*):** las consultas DNS tipicamente usan UDP por ser operaciones simples y rapidas. Si la respuesta no llega, la aplicacion reintenta. TCP se usa para transferencias de zona (large DNS responses) y para la replicacion entre servidores DNS.

**VoIP (*Voice over IP*):** la voz en tiempo real no puede tolerar la latencia de retransmision de TCP. Si un paquete se pierde, simplemente se omite (se nota como un clic o corte breve). TCP retransmitiria datos obsoletos que ya no son utiles para una conversacion en tiempo real.

**Streaming de video:** como VoIP, el video en tiempo real (Zoom,Teams,Google Meet) prefiere UDP. Los codecs modernos (H.264, H.265) pueden reconstruir frames perdidos a partir de frames circundantes.

**Juegos en linea:** los juegos requieren baja latencia absoluta. UDP permite enviar actualizaciones de estado sin esperar confirmaciones. TCP introduciria latencia inaceptable en juegos de accion en tiempo real.

**DHCP:** utiliza UDP en los puertos 67 (servidor) y 68 (cliente) porque el cliente no tiene direccion IP valida al momento de la solicitud, lo que hace imposible una conexion TCP.

## Cuando NO usar UDP

UDP no es adecuado para:

- **Transferencia de archivos:** cada byte perdido corrompe el archivo. FTP, SCP y SFTP usan TCP.
- **Correo electronico:** SMTP requiere entrega garantizada de cada mensaje.
- **Web:** HTTP/1.1 y HTTP/2 usan TCP. HTTP/3 (QUIC) usa UDP pero implementa fiabilidad a nivel de aplicacion.
- **Bases de datos:** las consultas SQL requieren integridad de datos.

> UDP es el protocolo del "mejor esfuerzo" (*best effort*): envia los datos y confia en que llegarán. Para aplicaciones que necesitan fiabilidad, TCP es la eleccion correcta. Para aplicaciones que priorizan velocidad y pueden tolerar (o gestionar) la perdida de datos, UDP es la plataforma ideal sobre la que construir protocolos de aplicacion eficientes.

