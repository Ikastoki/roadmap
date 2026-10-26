# Herramientas de diagnostico de la capa de transporte

El diagnostico de problemas en la capa de transporte requiere herramientas que permitan inspeccionar conexiones activas, puertos abiertos y trafico de red en tiempo real. Estas herramientas son esenciales para identificar cuellos de botella, conexiones fallidas, fugas de puertos y problemas de rendimiento.

Las herramientas de diagnostico se dividen en dos categorias: herramientas de inspeccion de estado (que muestran conexiones activas y puertos abiertos) y herramientas de captura de trafico (que permiten analizar el contenido de los paquetes).

## netstat y ss

**netstat** es la herramienta tradicional para inspeccionar conexiones de red, tablas de enrutamiento e interfazas. **ss** es su reemplazo moderno, mas rapido y con mas funcionalidades.

```bash
# Ver todas las conexiones TCP activas
ss -tan

# Ver solo puertos en escucha (listening)
ss -tuln

# Ver conexiones con procesos asociados
ss -tulnp

# Ver estadisticas de conexiones
ss -s

# Filtrar por puerto especifico
ss -tan | grep :443
```

Salida tipica de `ss -tuln`:

```
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
LISTEN  0       128     0.0.0.0:22           0.0.0.0:*
LISTEN  0       511     0.0.0.0:80           0.0.0.0:*
LISTEN  0       128     0.0.0.0:443          0.0.0.0:*
LISTEN  0       128    [::]:22              [::]:*
```

Los campos importantes son:

- **State:** estado de la conexion (LISTEN, ESTAB, TIME-WAIT, CLOSE-WAIT).
- **Recv-Q:** datos recibidos pendientes de ser leidos por la aplicacion.
- **Send-Q:** datos enviados pendientes de ser confirmados por el receptor.
- **Local Address:** IP y puerto local.

## lsof

`lsof` (*List Open Files*) muestra los archivos abiertos por un proceso, incluyendo sockets de red. Es util para identificar que proceso esta usando un puerto especifico.

```bash
# Ver procesos que usan el puerto 80
sudo lsof -i :80

# Ver todas las conexiones de red de un proceso
lsof -i -p $(pidof nginx)

# Ver conexiones TCP establecidas
sudo lsof -i TCP -sTCP:ESTABLISHED
```

## tcpdump y Wireshark

**tcpdump** es una herramienta de captura de paquetes basada en linea de comandos que permite filtrar y analizar trafico de red en tiempo real. Es util para diagnostico rapido sin necesidad de interfaz grafica.

```bash
# Capturar trafico en la interfaz eth0
sudo tcpdump -i eth0

# Capturar solo trafico TCP en el puerto 80
sudo tcpdump -i eth0 tcp port 80

# Capturar y guardar en archivo para analisis posterior
sudo tcpdump -i eth0 -w captura.pcap

# Filtrar por host especifico
sudo tcpdump -i eth0 host 192.168.1.100

# Capturar solo los primeros 10 paquetes
sudo tcpdump -i eth0 -c 10
```

**Wireshark** es la herramienta grafica de captura y analisis de paquetes mas completa. Ofrece filtros avanzados, reconstitucion de sesiones TCP, analisis de protocolos y estadisticas de trafico. Es la herramienta de referencia para el analisis de trafico de red.

```
Filtros utiles en Wireshark:
tcp.port == 443           # Trafico HTTPS
ip.addr == 10.0.0.5       # Todo el trafico a/desde una IP
tcp.flags.syn == 1        # Solo paquetes SYN
dns                        # Solo trafico DNS
http                       # Solo trafico HTTP
tcp.analysis.retransmission # Retransmisiones TCP
```

## Segmentacion y reensamblado

Cuando los datos de una aplicacion son mayores que la MTU del enlace, TCP segmenta los datos en multiples segmentos. El receptor reensambla los segmentos en el orden correcto antes de entregarlos a la aplicacion.

```bash
# Ver la MTU de una interfaz
ip link show eth0 | grep mtu

# Fragmentacion en IPv4 (los routers pueden fragmentar)
# IPv6 no permite fragmentacion en routers; solo el host origen
```

La segmentacion puede causar problemas de rendimiento si se produce en el camino: cada fragmento adicional significa mas cabeceras IP, mas procesamiento en routers y mayor probabilidad de perdida (si un fragmento se pierde, se reensamblan todos).

> Las herramientas de diagnostico son los ojos del administrador de redes en la capa de transporte. `ss` y `lsof` proporcionan una vision instantanea del estado de las conexiones; `tcpdump` y Wireshark revelan el contenido real del trafico. Dominar estas herramientas es indispensable para diagnosticar y resolver problemas de red en entornos de produccion.

