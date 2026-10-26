# TCP (Transmission Control Protocol)

TCP es el protocolo de transporte orientado a conexion y confiable que domina las comunicaciones en internet. Proporciona transferencia de datos ordenada, sin errores y con control de flujo y congestion. Es el protocolo elegido para aplicaciones que requieren fiabilidad: web (HTTP/HTTPS), correo (SMTP), transferencia de archivos (FTP) y acceso remoto (SSH).

TCP fue disenado por Vint Cerf y Bob Kahn en 1974 y se ha convertido en uno de los protocolos mas importantes de la historia de la informatica. Su diseno equilibrado entre fiabilidad y rendimiento lo ha mantenido relevante durante mas de 4 decadas.

## Handshake de 3 vias

Antes de transferir datos, TCP establece una conexion mediante un proceso de tres pasos:

```
Cliente                    Servidor
  │                          │
  │──── SYN (seq=100) ──────→│  1. Cliente propone conexion
  │                          │
  │←── SYN-ACK (seq=300,ack=101) ──│  2. Servidor acepta y propone
  │                          │
  │──── ACK (ack=301) ──────→│  3. Cliente confirma
  │                          │
  │     Conexion establecida │
```

**SYN (*Synchronize*):** el cliente envia un paquete SYN con su numero de secuencia inicial, indicando que desea establecer una conexion.

**SYN-ACK (*Synchronize-Acknowledge*):** el servidor responde con SYN-ACK, aceptando la peticion y proponiendo su propio numero de secuencia.

**ACK (*Acknowledge*):** el cliente confirma con un ACK, completando el establecimiento de la conexion.

El handshake de 3 vias garantiza que ambas partes estan listas para enviar y recibir datos, y que los numeros de secuencia estan sincronizados.

## Numeros de secuencia y acuse de recibo

Cada byte de datos enviado por TCP tiene un numero de secuencia unico. El receptor utiliza estos numeros para:

- **Reordenar segmentos:** si los segmentos llegan fuera de orden (commun en redes con multiples rutas), el receptor los reordena usando los numeros de secuencia.
- **Detectar perdidas:** si falta un numero de secuencia, el receptor sabe que un segmento se perdio y solicita retransmision.
- **Eliminar duplicados:** si un segmento llega dos veces (por retransmision innecesaria), el receptor lo descarta.

El campo *acknowledgment number* indica el proximo byte esperado, confirmando implicitamente la recepcion de todos los bytes anteriores.

## Ventana deslizante

El control de flujo en TCP se implementa con una **ventana deslizante**: el receptor indica al emisor cuantos bytes puede enviar antes de recibir un nuevo ACK. La ventana se actualiza con cada ACK recibido, permitiendo al emisor ajustar su ritmo de transmision a la capacidad del receptor.

```
Emisor                          Receptor
|--- Ventana de envio (N bytes) ---|
|  Seq 100  Seq 101  Seq 102 ...  |
        │                          │
        │──── ACK, window=M ──────→│
        │                          │
        |--- Ventana ajustada ---|
```

Si la ventana llega a cero, el emisor suspende la transmision y periodicamente envia segmentos de prueba (*window probe*) para verificar si la ventana se ha abierto.

## Control de congestion

El control de congestion evita que TCP sature la red enviando datos mas rapidos de los que la red puede transportar. TCP utiliza un conjunto de algoritmos para adaptar la velocidad de transmision:

- **Slow Start:** el emisor comienza enviando pocos datos y duplica la velocidad con cada ACK, creciendo exponencialmente hasta detectar perdidas.
- **Congestion Avoidance:** una vez alcanzado el umbral de congestion, la velocidad crece linealmente.
- **Fast Retransmit:** si el emisor recibe 3 ACKs duplicados, retransmite inmediatamente sin esperar timeout.
- **Fast Recovery:** despues de una perdida detectada por Fast Retransmit, reduce la ventana a la mitad y continua con Congestion Avoidance.

## Terminacion de conexion (4 vías)

La terminacion ordenada de una conexion TCP utiliza un proceso de cuatro pasos:

```
Cliente                    Servidor
  │                          │
  │──── FIN ────────────────→│  1. Cliente quiere cerrar
  │←──── ACK ────────────────│  2. Servidor confirma
  │←──── FIN ────────────────│  3. Servidor tambien quiere cerrar
  │──── ACK ────────────────→│  4. Cliente confirma
  │                          │
  │     Conexion cerrada     │
```

> TCP es un protocolo sofisticado que equilibra fiabilidad, rendimiento y utilizacion de la red. Su diseno ha resistido el paso del tiempo porque resuelve problemas fundamentales de la comunicacion de datos. Comprender TCP es esencial para diagnosticar problemas de rendimiento de red, ya que la mayoria de las aplicaciones dependen de el para la transferencia fiable de datos.

