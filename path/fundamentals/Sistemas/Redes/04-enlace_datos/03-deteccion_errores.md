# Deteccion y correccion de errores

La transmision de datos a traves de un medio fisico esta sujeta a errores: ruido electromagneticas, atenuacion de la senal, interferencias y otras perturbaciones pueden corromper los bits durante el viaje. La capa de enlace de datos implementa tecnicas para detectar y, en algunos casos, corregir estos errores, garantizando la integridad de los datos recibidos.

Es fundamental distinguir entre deteccion y correccion: detectar un error significa saber que los datos estan corruptos pero no saber cuales bits estan mal; corregir un error significa identificar y restaurar los bits incorrectos sin necesidad de retransmision.

## Tecnicas de deteccion

### Paridad

El bit de paridad es la tecnica mas simple de deteccion de errores. Se anade un bit adicional a cada grupo de datos de tal forma que el numero total de bits con valor 1 sea par (paridad par) o impar (paridad impar).

```
Datos: 1011001 (5 unos, impar)
Paridad par: 1011001 1 → total 6 unos (par)
Paridad impar: 1011001 0 → total 5 unos (impar)
```

Si un bit se corrompe durante la transmision, la paridad ya no coincide y se detecta el error. Limitacion: no detecta errores en numero par de bits (si cambian 2 bits, la paridad sigue siendo correcta).

### Checksum

El checksum es una suma de los campos de datos que se envia junto con ellos. El receptor recalcula la suma y la compara con la recibida. Si coinciden, los datos son correctos; si no, se han producido errores. Se utiliza en IP, TCP, UDP y otros protocolos.

```python
# Calculo simplificado de checksum (IP usa uno mas complejo)
datos = [0x45, 0x00, 0x00, 0x3C, 0x1C, 0x46, 0x40, 0x00,
         0x40, 0x06, 0x00, 0x00, 0xC0, 0xA8, 0x01, 0x01,
         0xC0, 0xA8, 0x01, 0x02]
suma = sum(datos) & 0xFFFF
checksum = ~suma & 0xFFFF
```

### CRC (Cyclic Redundancy Check)

El CRC es la tecnica mas utilizada en capa de enlace. Utiliza una operacion de division polinomica para calcular un valor de 32 bits (CRC-32 en Ethernet) que se coloca en el FCS de la trama.

El CRC es mucho mas robusto que el checksum para detectar errores multiples, errores de rafaga (*burst errors*) y errores que afectan a campos adyacentes. Ethernet utiliza CRC-32 que puede detectar el 99.999999954% de los errores de rafaga de hasta 32 bits.

## Tecnicas de correccion

### Codigo de Hamming

El codigo de Hamming anade bits de paridad calculados de forma que permiten no solo detectar sino tambien corregir errores de un solo bit. Para datos de 8 bits, se necesitan 4 bits de paridad (position 1, 2, 4, 8). Si un bit se corrompe, los bits de paridad revelan exactamente cual es el bit erroneo.

Hamming es util en memoria RAM (ECC) pero tiene overhead significativo para transmision de red: por cada 8 bits de datos, necesitaria 4 bits adicionales de paridad.

### ARQ (Automatic Repeat reQuest)

ARQ no corrige errores directamente sino que solicita una retransmision de los datos erroneos. Es el mecanismo utilizado en TCP y en muchos protocolos de capa de enlace:

**Stop-and-Wait ARQ:** el emisor envia una trama y espera el ACK antes de enviar la siguiente. Si no recibe ACK en un tiempo determinado, retransmite. Simple pero ineficiente.

**Go-Back-N ARQ:** el emisor puede enviar multipes tramas sin esperar ACK. Si una trama se pierde o corrompe, retransmite esa trama y todas las posteriores (*go back to N*). Mas eficiente que Stop-and-Wait.

**Selective Repeat ARQ:** similar a Go-Back-N pero solo retransmite las tramas que fallaron, no las posteriores. Mas eficiente pero requiere mas buffer en el receptor.

## Diferencia entre deteccion y correccion

| Aspecto | Deteccion | Correccion |
|---|---|---|
| Funcion | Identificar que hay un error | Identificar y corregir el error |
| Overhead | Bajo (FCS de 32 bits) | Alto (bits de paridad adicionales) |
| Accion tras error | Descartar trama, retransmitir | Corregir in situ |
| Uso comun | Ethernet, IP, TCP | Memoria ECC, comunicaciones espaciales |

> En la practica, las redes modernas combinan ambas aproximaciones: el FCS de la trama Ethernet detecta errores con alta fiabilidad, y TCP proporciona retransmision confiable a nivel de transporte. Los codigos de correccion de errores (FEC) se utilizan en medios donde las retransmisiones no son viables, como la comunicacion por satelite o las grabaciones opticas (CD/DVD/Blu-ray).

