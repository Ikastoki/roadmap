# Protocolos esenciales

Los protocolos de la capa de aplicacion son los que la mayoria de los usuarios y administradores de sistemas interactuan directamente. HTTP para la web, DNS para la resolucion de nombres, SMTP para el correo y SSH para el acceso remoto son los pilares que sustentan la experiencia diaria en internet.

Comprender estos protocolos no es solo cuestion de curiosidad tecnica: saber como funciona DNS te permite diagnosticar por que una pagina no carga; saber como funciona SMTP te ayuda a resolver problemas de correo; y saber como funciona SSH es esencial para la administracion remota de servidores.

## HTTP/HTTPS

**HTTP** (*Hypertext Transfer Protocol*) es el protocolo de la World Wide Web. El cliente (navegador) envia peticiones HTTP a un servidor web, que responde con paginas HTML, imagenes, scripts y otros recursos.

**Metodos principales:**

- **GET:** solicita un recurso. Es idempotente y seguro (no modifica datos).
- **POST:** envia datos al servidor para crear un recurso. No es idempotente.
- **PUT:** reemplaza un recurso completo. Es idempotente.
- **DELETE:** elimina un recurso.
- **PATCH:** modifica parcialmente un recurso.

**Codigos de estado mas comunes:**

- **200 OK:** la peticion fue exitosa.
- **301/302:** redireccion permanente/temporal.
- **404 Not Found:** el recurso no existe.
- **500 Internal Server Error:** error del servidor.
- **429 Too Many Requests:** rate limiting.

**HTTPS** es HTTP cifrado con TLS. Protege la confidencialidad e integridad de los datos en transito. El certificado X.509 del servidor autentica su identidad y permite el intercambio seguro de claves.

## DNS

**DNS** (*Domain Name System*) resuelve nombres de dominio a direcciones IP. Cuando tecleas `www.google.com` en tu navegador, DNS traduce ese nombre a la IP correspondiente (por ejemplo, 142.250.185.78).

La resolucion jerarquica:

```
Navegador → Cache local → Resolver del SO → Servidor DNS del ISP → Root DNS → TLD (.com) → Autoritativo
```

**Registros DNS comunes:**

- **A:** mapea nombre a IPv4.
- **AAAA:** mapea nombre a IPv6.
- **CNAME:** alias a otro nombre.
- **MX:** servidor de correo electronico.
- **NS:** servidor de nombres autoritativo.
- **TXT:** registros de texto (SPF, DKIM, verificacion de dominio).

```bash
# Consultas DNS
dig www.google.com
dig MX gmail.com
dig +short google.com
nslookup example.com
```

## SMTP, POP3 e IMAP

**SMTP** (*Simple Mail Transfer Protocol*) es el protocolo para enviar correo electronico. Opera en el puerto 587 (submission) o 25 (entre servidores). Los clientes de correo envian mensajes al servidor SMTP, que los reenvia al servidor de destino.

**POP3** (*Post Office Protocol v3*) descarga los correos del servidor al cliente y tipicamente los elimina del servidor. Puerto 110 (995 con TLS). Util cuando se accede al correo desde un unico dispositivo.

**IMAP** (*Internet Message Access Protocol*) mantiene los correos en el servidor y permite sincronizar el estado (leido/no leido, carpetas, etiquetas) entre multiples dispositivos. Puerto 143 (993 con TLS). Es el estandar actual para acceder al correo desde multiples dispositivos.

## SSH

**SSH** (*Secure Shell*) proporciona acceso remoto seguro a un servidor de comandos. Cifra toda la comunicacion, incluyendo la autenticacion y los comandos ejecutados.

```bash
# Conexion basica
ssh usuario@servidor

# Conexion con clave publica
ssh -i ~/.ssh/id_ed25519 usuario@servidor

# Port forwarding (tunel)
ssh -L 8080:localhost:80 usuario@servidor

# Tunnel remoto
ssh -R 9090:localhost:3000 usuario@servidor
```

SSH utiliza autenticacion por clave publica (Ed25519 recomendado sobre RSA) o por contrasena (menos seguro). El servidor almacena las claves publicas autorizadas en `~/.ssh/authorized_keys`.

## FTP, SFTP y FTPS

**FTP** (*File Transfer Protocol*) transfiere archivos entre cliente y servidor. Utiliza dos canales: control (puerto 21) y datos (puerto efimero). No cifra las credenciales ni los datos.

**SFTP** (*SSH File Transfer Protocol*) es FTP sobre SSH. Cifra toda la comunicacion. Es el recomendado para transferencias seguras. Puerto 22.

**FTPS** es FTP con TLS. Cifra la comunicacion pero es mas complejo de configurar que SFTP.

## APIs y protocolos modernos

- **REST:** arquitectura basada en HTTP/HTTPS que utiliza metodos HTTP para operaciones CRUD sobre recursos identificados por URIs.
- **gRPC:** framework RPC de alto rendimiento que utiliza HTTP/2 y Protocol Buffers. Ideal para comunicacion entre microservicios.
- **WebSockets:** protocolo que proporciona canales bidireccionales persistentes sobre HTTP. Util en chat en tiempo real, juegos y dashboards.

> Los protocolos de aplicacion son el punto de encuentro entre la red y el usuario final. Un conocimiento solido de estos protocolos permite diagnosticar problemas de conectividad, optimizar el rendimiento, implementar seguridad y disenar nuevas aplicaciones que se integren eficazmente con la infraestructura existente.

