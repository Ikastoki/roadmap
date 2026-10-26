# Puertos y sockets

Los puertos son la base de la multiplexacion en la capa de transporte: permiten que multiples aplicaciones en un mismo host compartan una unica direccion IP. Un **socket** es la combinacion de una direccion IP, un numero de puerto y un protocolo de transporte (TCP o UDP), y constituye el punto final de una comunicacion de red.

Los puertos y sockets son conceptos que conectan el mundo de las redes (IP, paquetes) con el mundo de las aplicaciones (archivos, flujos de datos). Las aplicaciones abren sockets para enviar y recibir datos, y el sistema operativo gestiona la distribucion de datos a los procesos correctos segun los puertos.

## Rangos de puertos

Los puertos de 16 bits proporcionan un espacio de 65536 numeros, divididos en tres rangos:

**Puertos bien conocidos (0-1023):** reservados para servicios de sistema y protocolos estandarizados. Solo pueden ser abiertos por procesos con privilegios de root. Ejemplos: HTTP (80), HTTPS (443), SSH (22), DNS (53), SMTP (25), FTP (21), Telnet (23), POP3 (110), IMAP (143).

**Puertos registrados (1024-49151):** asignados por IANA para aplicaciones especificas pero sin restriccion de privilegios. Ejemplos: MySQL (3306), PostgreSQL (5432), RDP (3389), VNC (5900).

**Puertos dinamicos/efimeros (49152-65535):** asignados automaticamente por el sistema operativo cuando una aplicacion necesita un puerto efimero para una conexion saliente. No estan reservados y pueden ser reutilizados.

```bash
# Ver puertos en uso
ss -tuln

# Ver que proceso usa un puerto
sudo lsof -i :80
sudo ss -tulnp | grep :80
```

## Socket = IP + puerto + protocolo

Un socket es la direccion logica de un punto final de comunicacion. La combinacion completa que identifica una conexion unica es:

```
(IP_origen, puerto_origen, protocolo) <-> (IP_destino, puerto_destino, protocolo)

Ejemplo de conexion TCP:
(192.168.1.10, 54321, TCP) <-> (10.0.0.5, 22, TCP)
```

Millones de conexiones pueden coexistir en un mismo servidor, cada una identificada por un socket unico. Un servidor web puede tener 10,000 conexiones TCP simultaneas todas usando el puerto 443, pero cada una desde un puerto efimero diferente del cliente.

## Tipo de sockets

Los sistemas operativos Unix/Linux soportan varios dominios de sockets:

- **AF_INET:** sockets IPv4 (el mas comun).
- **AF_INET6:** sockets IPv6.
- **AF_UNIX (AF_LOCAL):** sockets de dominio Unix para comunicacion en la misma maquina. Mas rapidos que TCP/IPC ya que no traversan la pila de red.
- **AF_BLUETOOTH:** sockets para comunicacion Bluetooth.

Y varios tipos de sockets:

- **SOCK_STREAM:** orientado a conexion (TCP).
- **SOCK_DGRAM:** sin conexion (UDP).
- **SOCK_RAW:** acceso directo a la capa de red (para herramientas de diagnostico como ping o traceroute).

## Ejemplo: ciclo de vida de un socket TCP

```python
# Servidor
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.bind(('0.0.0.0', 8080))          # Asociar a puerto 8080
sock.listen(5)                          # Escuchar conexiones
conn, addr = sock.accept()              # Aceptar conexion entrante
data = conn.recv(1024)                  # Recibir datos
conn.send(b'Respuesta')                # Enviar datos
conn.close()                            # Cerrar conexion
sock.close()
```

```python
# Cliente
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('10.0.0.5', 8080))       # Conectar al servidor
sock.send(b'Hola servidor')            # Enviar datos
data = sock.recv(1024)                 # Recibir respuesta
sock.close()
```

> El concepto de socket es el puente entre las redes y las aplicaciones. Las bibliotecas de sockets (BSD sockets) son la API estandar para programar aplicaciones de red en Unix/Linux, y la mayoria de los protocolos de red (HTTP, SSH, SMTP) se implementan sobre sockets TCP o UDP.

