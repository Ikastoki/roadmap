# Sincronizacion en sesiones

La sincronizacion en la capa de sesion permite a las aplicaciones mantener puntos de referencia dentro de una comunicacion larga, de modo que si se produce un fallo, la sesion pueda reanudarse desde el ultimo punto de sincronizacion valido en lugar de comenzar desde cero.

Este mecanismo es particularmente util en transferencias de archivos grandes, videollamadas y sesiones interactivas donde las interrupciones son frecuentes y la retransmision completa de datos seria ineficiente.

## Puntos de sincronizacion

Un punto de sincronizacion (*sync point*) es una marca logica dentro del flujo de datos que indica un estado conocido y verificado. Ambas partes de la sesion registran los puntos de sincronizacion y pueden acordar reanudar desde ellos en caso de fallo.

```
Cliente                    Servidor
  │──── Datos 1-1000 ──────→│
  │──── Datos 1001-2000 ───→│
  │──── Sync Point (2000) ─→│
  │──── Datos 2001-3000 ───→│
  │                          │ ← Fallo en la red
  │──── Solicitar reanudacion desde 2000 ──→│
  │──── Datos 2001-3000 ───→│  (retransmision parcial)
```

## Checkpoints para recuperacion

Los *checkpoints* son puntos de sincronizacion que la aplicacion establece en intervalos regulares o en momentos semanticamente significativos (por ejemplo, despues de cada 100 archivos transferidos en una sincronizacion de directorio).

En transferencias de archivos, los checkpoints permiten reanudar desde el ultimo bloque verificado. Sin checkpoints, una transferencia de 10 GB que falla al 99% tendria que reiniciarse completamente. Con checkpoints cada 100 MB, solo se retransmitirian los ultimos 100 MB.

## Reanudacion de sesiones tras fallos

La reanudacion de sesiones varia segun el protocolo y la aplicacion:

- **TCP:** no implementa reanudacion a nivel de sesion; si la conexion TCP se pierde, debe establecerse una nueva. Sin embargo, TCP mantiene buffers de retransmision que pueden recuperar datos pendientes si la conexion se recupera antes del timeout.
- **HTTP/1.1:** soporta rangos (*byte ranges*) que permiten reanudar descargas desde un punto especifico. El servidor responde con el codigo 206 Partial Content.
- **SSH:** mantiene el estado de la sesion y puede reconectar si la conexion se interrumpe temporalmente (*connection keepalive*).
- **SFTP:** soporta reanudacion de transferencias con el comando `reget`.

> La sincronizacion es un concepto que en TCP/IP se implementa principalmente a nivel de aplicacion, no como una capa separada. Cada protocolo resuelve el problema de la recuperacion ante fallos a su manera: HTTP con rangos byte, SSH con reconexion, rsync con checksums de bloques. Entender el concepto de sincronizacion ayuda a seleccionar y configurar protocolos que se recuperan eficientemente ante interrupciones.

