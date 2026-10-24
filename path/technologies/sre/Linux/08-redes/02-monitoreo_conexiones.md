# Monitoreo de conexiones de red

Un servidor Linux rara vez esta aislado. constantemente establece, mantiene y cierra conexiones TCP con clientes, otros servidores y servicios externos. Saber que conexiones estan activas, que puertos estan abiertos y que procesos las sostienen es una competencia basica para la seguridad y el diagnostico. Linux proporciona tres herramientas fundamentales para esta tarea: `ss`, `netstat` y `lsof`.

## ss: la herramienta moderna

`ss` (socket statistics) es la herramienta actual para inspeccionar sockets de red, reemplazando a `netstat` en las distribuciones modernas. Su velocidad y flexibilidad la convierten en la opcion preferida:

```bash
ss -t                  # Muestra solo sockets TCP
ss -u                  # Muestra solo sockets UDP
ss -l                  # Muestra solo sockets en escucha (listening)
ss -a                  # Muestra todos los sockets (escucha + establecidos)
ss -n                  # Sin resolucion de nombres (mas rapido)
ss -p                  # Muestra el proceso asociado al socket
```

Las opciones se combinan para obtener vistas especificas:

```bash
ss -tlnp               # Puertos TCP en escucha, con numeros y procesos
ss -tunap              # TCP y UDP, todos los estados, con procesos
ss -tn state established  # Solo conexiones TCP establecidas
ss -tn state listening     # Solo sockets en escucha
ss -t '( dport = :443 or sport = :443 )'  # Filtra por puerto
```

La salida muestra las direcciones locales y remotas, el estado del socket (LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT, etc.) y, con `-p`, el PID y nombre del proceso. El estado TIME_WAIT, por ejemplo, es normal en conexiones TCP que se cierran, pero un numero excesivo de ellos puede agotar los puertos efimeros.

> `ss` lee la informacion directamente del subsistema de sockets del kernel a traves del pseudofichero `/proc/net/tcp`, lo que la hace significativamente mas rapida que `netstat`.

## netstat: la herramienta clasica

`netstat` sigue disponible en la mayoria de sistemas y ofrece una salida similar a `ss`. Aunque considerada obsoleta, su sintaxis sigue siendo ampliamente conocida:

```bash
netstat -tlnp          # Puertos TCP en escucha
netstat -tunap         # Todas las conexiones, TCP y UDP
netstat -i             # Estadisticas de interfaces de red
netstat -s             # Estadisticas de protocolo (paquetes enviados/recibidos, errores)
netstat -rn            # Tabla de enrutamiento (equivalente a ip route)
```

`netstat -s` es particularmente util para detectar problemas a nivel de protocolo: retransmisiones TCP elevadas, segmentos descartados, paquetes con errores de checksum o paquetes reassembled failed en IP.

## lsof -i: asociar puertos a procesos

Mientras que `ss` y `netstat` muestran el proceso asociado a un socket, `lsof` (list open files) permite buscar archivos abiertos, y en Linux los sockets de red tambien son archivos. La opcion `-i` filtra por sockets de internet:

```bash
lsof -i                # Todos los sockets de red abiertos
lsof -i :80            # Procesos que usan el puerto 80
lsof -i TCP:22         # Conexiones TCP en el puerto 22
lsof -i @192.168.1.1   # Conexiones con una IP concreta
lsof -i -s TCP:LISTEN  # Solo sockets en escucha
lsof -i -P -n          # Sin resolucion de nombres (mas rapido)
lsof -i -u usuario     # Conexiones de un usuario especifico
```

`lsof -i :puerto` es la forma mas rapida de responder a la pregunta "que proceso esta usando este puerto?". Esto es especialmente util cuando un servicio no arranca porque el puerto ya esta ocupado, o cuando se sospecha que un proceso no autorizado esta escuchando en un puerto determinado.

```bash
# Ejemplo de busqueda de un proceso en el puerto 8080
lsof -i :8080
COMMAND   PID     USER   FD   TYPE DEVICE SIZE NODE NAME
java     2345    app    56u  IPv4 12345       TCP *:8080 (LISTEN)
```

> La trfada de herramientas (ss, netstat, lsof) permite triangular informacion: `ss` para velocidad y volumen, `netstat` para estadisticas de protocolo, `lsof` para relacionar puertos con procesos concretos. Las tres leen del mismo subsistema del kernel, por lo que la informacion es consistente.

