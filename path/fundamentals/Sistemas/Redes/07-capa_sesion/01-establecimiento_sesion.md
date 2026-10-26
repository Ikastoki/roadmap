# Establecimiento, mantenimiento y cierre de sesion

La capa de sesion gestiona el dialogo estructurado entre dos aplicaciones que se comunican a traves de la red. Aunque en TCP/IP las funciones de la capa de sesion suelen ser asumidas por TCP (capa 4) o por las propias aplicaciones, el concepto de sesion sigue siendo fundamental para comprender como se gestionan las comunicaciones de larga duracion como videollamadas, transferencias de archivos y sesiones interactivas.

Una sesion es una comunicacion logica entre dos aplicaciones que se extiende a lo largo del tiempo y que puede incluir multiples intercambios de datos, sincronizaciones y controles de dialogo.

## Que es una sesion en redes

Una sesion de red es el contexto mantenido entre dos aplicaciones durante una comunicacion. Incluye el estado de la comunicacion: que datos se han enviado, que se han recibido, que posicion se encuentra en una transferencia y que operaciones pendientes quedan.

A diferencia de un paquete aislado (que no tiene estado), una sesion mantiene informacion que permite retomar la comunicacion despues de interrupciones, verificar la integridad del intercambio y coordinar las acciones entre ambas partes.

## Control de dialogo

El control de dialogo determina quien puede transmitir y cuando:

**Half-duplex alternante:** solo un lado puede transmitir en cada momento. El dialogo alterna entre emisor y receptor, similar a una conversacion por walkie-talkie. Util en sesiones donde el intercambio es secuencial: primero el cliente envia una peticion, luego el servidor responde.

**Full-duplex simultaneo:** ambas partes pueden transmitir y recibir al mismo tiempo. Util en videollamadas, juegos en linea y sesiones interactivas donde ambos flujos de datos son independientes.

TCP permite full-duplex desde el establecimiento de la conexion. La capa de sesion puede implementar control de dialogo adicional sobre la base TCP para aplicaciones que lo requieran.

## Mecanismos de cierre ordenado

El cierre ordenado de una sesion garantiza que todos los datos pendientes se han entregado correctamente antes de terminar la comunicacion. En TCP, esto se logra con el proceso de terminacion de 4 vias (FIN, ACK, FIN, ACK).

Las aplicaciones pueden implementar cierres mas sofisticados:

- **Cierre simetrico:** cada lado cierra su flujo de envio independientemente. El servidor puede indicar "no tengo mas datos para ti" mientras sigue recibiendo datos del cliente.
- **Cierre abrupto:** si una aplicacion cierra la conexion sin el protocolo de terminacion ordenada, el receptor recibe un RST (reset) y sabe que la sesion termino inesperadamente.

> En TCP/IP, la capa de sesion no se implementa como una capa separada. TCP gestiona el estado de la conexion, y las aplicaciones implementan la logica de sesion que necesitan. Sin embargo, entender el concepto de sesion es util para diagnosticar problemas como conexiones colgadas (sockets en estado CLOSE-WAIT), sesiones expiradas o perdida de estado en aplicaciones.

