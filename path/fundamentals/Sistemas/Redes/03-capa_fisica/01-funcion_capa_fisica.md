# Funcion de la capa fisica

La capa fisica es la base de toda la comunicacion en red. Es la capa mas cercana al hardware y su unica responsabilidad es transmitir y recibir bits a traves de un medio fisico. No interpreta los bits, no los organiza en estructuras, no los dirige a ningun destino especifico: simplemente los convierte en senales electricas, opticas o electromagneticas y las envia por el canal.

Aunque suena simple, la capa fisica resuelve problemas fundamentales: como representar un 0 y un 1 con voltajes electricos, como sincronizar emisor y receptor para que ambos sepan cuando empieza y termina un bit, y como maximizar la cantidad de informacion que puede viajar por un canal con un ancho de banda limitado.

## Que significa transmitir bits

Cuando un ordenador envia datos a traves de la red, la capa fisica toma los bits de la trama y los convierte en una senal fisica. En Ethernet cableado, un bit 1 se representa como un voltaje alto y un bit 0 como un voltaje bajo. En fibra optica, un 1 es un pulso de luz y un 0 es la ausencia de luz. En Wi-Fi, los bits se codifican variando la amplitud, frecuencia y fase de una onda de radio.

La velocidad de transmision se mide en bits por segundo (bps). Una conexion Ethernet de 1 Gbps transmite mil millones de bits por segundo, lo que significa que la senal electrica cambia de estado mil millones de veces por segundo.

## Bitrate y baud rate

Es importante distinguir entre bitrate y baud rate:

**Bitrate:** numero de bits transmitidos por segundo. Es la medida mas comun de velocidad de red. Una conexion de 100 Mbps tiene un bitrate de 100 millones de bits por segundo.

**Baud rate (symbol rate):** numero de cambios de senal (simbolos) por segundo. Cada simbolo puede representar multiples bits dependiendo de la tecnica de modulacion utilizada. Por ejemplo, si cada simbolo representa 4 bits (como en QAM-16), un baud rate de 250 Mbps equivale a un bitrate de 1 Gbps.

La relacion entre bitrate y baud rate depende de la complejidad de la modulacion: a mayor numero de bits por simbolo, mayor bitrate con el mismo baud rate. Sin embargo, simbolos mas complejos son mas susceptibles al ruido.

## Que NO hace la capa fisica

La capa fisica no:

- **No direcciona:** no maneja direcciones MAC ni IP. No sabe a que dispositivo van dirigidos los bits.
- **No detecta errores:** no verifica si los bits se transmitieron correctamente. La deteccion de errores es responsabilidad de la capa de enlace.
- **No interpreta datos:** no distingue entre cabeceras, payload ni FCS. Para la capa fisica, todos los bits son iguales.
- **No controla el acceso al medio:** no determina que dispositivo puede transmitir en cada momento. Esto es responsabilidad de la capa de enlace.

## Elementos que controla la capa fisica

La capa fisica define y controla:

- **Voltaje:** los niveles de voltaje que representan bits 0 y 1. En RS-232, por ejemplo, -15V a -3V representa un 1 logico y +3V a +15V un 0 logico.
- **Modulacion:** la tecnica utilizada para codificar bits en la senal fisica (ASK, FSK, PSK, QAM).
- **Sincronizacion:** mecanismos que permiten al receptor saber cuando comienza y termina cada bit o simbolo.
- **Velocidad de transmision:** la tasa a la que se envian bits por el canal.
- **Topologia fisica:** la disposicion de los dispositivos en el medio.

> La capa fisica es donde la informatica se encuentra con la ingenieria electronica. Los avances en modulacion, codificacion de senales y materiales de transmision (de cobre a fibra optica) han multiplicado la capacidad de las redes en las ultimas decadas, desde 10 Mbps de Ethernet clasico hasta los Tbps de las redes opticas modernas.

