# Herramientas de diagnostico

Cuando una aplicacion de red falla, el primer paso no es revisar el codigo sino determinar que esta ocurriendo realmente en la capa de transporte. Las herramientas de diagnostico permiten inspeccionar conexiones activas, analizar trafico, medir latencias y reproducir peticiones manualmente. Conocerlas es tan importante como dominar la API de red.

## netstat / ss

`netstat` y su sustituto moderno `ss` muestran conexiones de red activas, puertos en escucha y estadisticas de protocolo.

```bash
# Conexiones activas
netstat -tunapl

# Solo puertos en escucha
netstat -tlnp

# Con ss (mas rapido y completo)
ss -tuna
ss -tlnp
```

- `-t`: TCP, `-u`: UDP, `-n`: direcciones numericas, `-a`: todas, `-l`: listening, `-p`: proceso asociado.

```bash
# Verificar que un servidor esta escuchando en puerto 8080
ss -tlnp | grep 8080
```

> Si una aplicacion Java no arranca porque "el puerto ya esta en uso", `ss -tlnp | grep puerto` muestra que proceso lo esta ocupando, incluyendo el PID y el nombre del proceso.

## curl

`curl` es el cliente HTTP/HTTPS mas versatil en linea de comandos. Permite probar APIs REST, inspeccionar cabeceras y simular cualquier metodo HTTP.

```bash
# GET basico
curl http://localhost:8080/api/usuarios

# Mostrar cabeceras de respuesta
curl -i http://localhost:8080/api/usuarios

# Solo cabeceras (sin cuerpo)
curl -I http://localhost:8080/api/usuarios

# POST con JSON
curl -X POST http://localhost:8080/api/usuarios \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Ana","email":"ana@example.com"}'

# Seguir redirecciones
curl -L http://ejemplo.com

# Timeout y verbose
curl -v --connect-timeout 5 --max-time 10 http://ejemplo.com

# Con autenticacion Basic
curl -u usuario:password http://api.ejemplo.com/privado

# Con certificado SSL inseguro (desarrollo)
curl -k https://localhost:8443/api
```

## Wireshark

Wireshark es el analizador de protocolos de red mas completo. Captura paquetes en tiempo real y permite inspeccionar cada campo de cada protocolo (TCP, HTTP, TLS, DNS, etc.). Es la herramienta definitiva cuando se necesita entender exactamente que datos viajan por la red.

**Casos de uso tipicos:**

- Verificar el handshake TLS (versiones, cifradores, certificados).
- Analizar latencias de red (tiempos entre SYN, SYN-ACK, ACK).
- Depurar problemas de reinicio de conexion (RST).
- Confirmar que las cabeceras HTTP enviadas son correctas.

```bash
# Filtrar trafico HTTP a un host especifico
http.host == "api.ejemplo.com"

# Filtrar por puerto
tcp.port == 8080

# Filtrar handshake TLS
tls.handshake.type == 1
```

> Wireshark no puede capturar trafico en sistemas de produccion sin permisos de superusuario. En Linux, se necesita `sudo` o el grupo `wireshark`. Para analisis rapidos desde terminal, `tshark` (version CLI de Wireshark) es una alternativa eficiente.

## Postman

Postman es un entorno de desarrollo de APIs con interfaz grafica. Permite construir peticiones HTTP complejas, guardar colecciones, generar documentacion y automatizar pruebas.

**Caracteristicas clave:**
- Editor visual de peticiones con historial.
- Variables de entorno para switchear entre desarrollo, staging, produccion.
- Scripting pre-request y post-response en JavaScript.
- Generacion automatica de codigo en varios lenguajes (Java, Python, cURL).

## telnet

`telnet` es la herramienta mas basica para probar conectividad TCP. No realiza negociacion de protocolo; simplemente abre un socket TCP y permite enviar texto directamente.

```bash
# Probar que un puerto esta abierto
telnet localhost 8080

# Enviar peticion HTTP manualmente
telnet api.ejemplo.com 80
GET / HTTP/1.1
Host: api.ejemplo.com

```

La respuesta del servidor se muestra directamente en la terminal. Si el puerto esta filtrado o cerrado, telnet muestra "Connection refused" o "Connection timed out".

> `telnet` solo funciona con texto plano. Para probar conexiones TLS/SSL se usa `openssl s_client`:
> ```bash
> openssl s_client -connect api.ejemplo.com:443 -tls1_3
> ```

## nc (netcat)

`nc` es una alternativa mas moderna a telnet que soporta UDP, escucha de puertos y transferencia de archivos.

```bash
# Probar puerto TCP
nc -zv localhost 8080

# Escuchar en un puerto (para pruebas)
nc -l -p 9999

# Enviar archivo
cat datos.json | nc localhost 8080
```

## Tabla resumen

| Herramienta | Uso principal | Nivel |
|---|---|---|
| `ss` / `netstat` | Puertos y conexiones | Transporte |
| `curl` | Peticiones HTTP | Aplicacion |
| Wireshark | Captura y analisis de paquetes | Todos |
| Postman | Desarrollo y testing de APIs | Aplicacion |
| `telnet` | Conectividad TCP basica | Transporte |
| `nc` | Depuracion TCP/UDP generica | Transporte |
| `openssl s_client` | Diagnostico TLS | Presentacion |
