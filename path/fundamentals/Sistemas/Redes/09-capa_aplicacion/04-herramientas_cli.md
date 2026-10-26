# Herramientas CLI de la capa de aplicacion

Las herramientas de linea de comandos son esenciales para diagnosticar problemas, probar servicios y gestionar comunicaciones de red desde la terminal. A diferencia de las interfaces graficas, las herramientas CLI proporcionan acceso directo a los protocolos, permiten automatizacion y son indispensables en servidores sin interfaz grafica.

Dominar estas herramientas es una de las habilidades mas valiosas para un administrador de sistemas o ingeniero de redes: permiten diagnosticar en segundos problemas que en una interfaz grafica requeririan multiples clics y navegacion por menus.

## curl

`curl` es la herramienta mas versatil para transferir datos con protocolos de red. Soporta HTTP, HTTPS, FTP, SCP, SFTP, SMTP, LDAP y decenas de protocolos mas.

```bash
# Peticion GET basica
curl https://api.ejemplo.com/users

# Ver cabeceras HTTP
curl -I https://ejemplo.com

# Peticion POST con JSON
curl -X POST https://api.ejemplo.com/users \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Ana", "email": "ana@ejemplo.com"}'

# Seguir redirecciones
curl -L https://ejemplo.com

# Verbose (mostrar negociacion TLS y cabeceras)
curl -v https://ejemplo.com

# Guardar respuesta en archivo
curl -o archivo.zip https://ejemplo.com/datos.zip

# Autenticacion basica
curl -u usuario:contrasena https://api.ejemplo.com/admin
```

## wget

`wget` es una herramienta de descarga que soporta reanudacion, descargas recursivas y autenticacion. Es mas robusto que curl para descargas de archivos grandes.

```bash
# Descargar un archivo
wget https://ejemplo.com/archivo.zip

# Descargar con nombre personalizado
wget -O backup.tar.gz https://ejemplo.com/backup.tar.gz

# Descarga recursiva de un sitio web
wget -r -np -l 2 https://ejemplo.com

# Reanudar descarga interrumpida
wget -c https://ejemplo.com/archivo_grande.iso

# Descarga en background
wget -b https://ejemplo.com/archivo.iso
```

## dig y nslookup

**dig** (*Domain Information Groper*) es la herramienta de consulta DNS mas completa y detallada.

```bash
# Consulta basica A
dig www.google.com

# Consulta MX (correo)
dig MX gmail.com

# Consulta inversa
dig -x 8.8.8.8

# Usar un servidor DNS especifico
dig @8.8.8.8 www.google.com

# Salida concisa
dig +short google.com

# Todos los registros
dig ANY ejemplo.com

# Ver el tiempo de respuesta
dig +stats +ttlid google.com
```

**nslookup** es una alternativa mas simple a dig.

```bash
nslookup www.google.com
nslookup -type=MX gmail.com
nslookup www.google.com 8.8.8.8
```

## telnet y nc (netcat)

**telnet** permite probar conexiones TCP a un puerto especifico. Es util para verificar si un servicio esta escuchando y accesible.

```bash
# Probar conexion HTTP
telnet localhost 80

# Probar conexion SSH
telnet servidor 22
```

**netcat** (`nc`) es la "navaja suiza" de las redes. Permite crear conexiones TCP/UDP arbitrarias, escuchar puertos y transferir datos.

```bash
# Escuchar en un puerto
nc -l 8080

# Conectar a un puerto
nc servidor 443

# Verificar si un puerto esta abierto
nc -zv servidor 80

# Escanear un rango de puertos
nc -zv servidor 20-100

# Transferir archivos
# Receptor:
nc -l 9999 > archivo_recibido.txt
# Emisor:
nc receptor 9999 < archivo_enviar.txt
```

## Servidores web locales para pruebas

```bash
# Servidor HTTP basico con Python
python3 -m http.server 8000
python3 -m http.server 8000 --directory /var/www

# Servidor HTTPS (requiere certificado)
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
python3 -c "
import http.server, ssl
s = http.server.HTTPServer(('0.0.0.0', 8443), http.server.SimpleHTTPRequestHandler)
s.socket = ssl.wrap_socket(s.socket, keyfile='key.pem', certfile='cert.pem', server_side=True)
s.serve_forever()
"
```

> Las herramientas CLI son la forma mas directa e eficiente de interactuar con los protocolos de red. Mientras que las interfaces graficas ocultan los detalles bajo capas de abstraccion, las herramientas CLI exponen el protocolo en toda su complejidad, permitiendo un diagnostico preciso y un control total sobre las comunicaciones.

