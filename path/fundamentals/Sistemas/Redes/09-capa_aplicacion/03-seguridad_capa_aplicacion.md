# Seguridad en la capa de aplicacion

La seguridad en la capa de aplicacion protege los datos y las comunicaciones contra interceptacion, manipulacion y suplantacion. TLS/SSL cifra las comunicaciones; los certificados autentican la identidad de los servidores; y las practicas de desarrollo seguro previenen vulnerabilidades en las propias aplicaciones.

La seguridad en esta capa es donde mas impacto tiene el error humano: un servidor HTTPS mal configurado, un certificado caducado o una contrasena debil pueden comprometer toda la comunicacion, independientemente de lo robustas que sean las capas inferiores.

## TLS/SSL

TLS (*Transport Layer Security*) es el protocolo que cifra las comunicaciones en HTTPS, correo electronico, VPNs y muchas otras aplicaciones. El predecessor (SSL, *Secure Suite Layers*) esta obsoleto y no debe utilizarse.

El proceso TLS combina cifrado asimetrico (para el intercambio de claves) y cifrado simetrico (para la transmision de datos):

1. El cliente envia un *ClientHello* con los algoritmos que soporta.
2. El servidor responde con un *ServerHello*, eligiendo los algoritmos y enviando su certificado.
3. El cliente verifica el certificado contra las CA de confianza.
4. Se intercambian claves para derivar una clave simetrica compartida.
5. Toda la comunicacion posterior se cifra con AES o ChaCha20.

TLS 1.3 es la version actual recomendada: elimina algoritmos inseguros, reduce el handshake a 1 RTT y soporta *0-RTT* para reanudaciones rapidas.

## Certificados digitales

Un certificado digital es un documento electronico que vincula una entidad (dominio, organizacion) con una clave publica, firmado por una autoridad de certificacion (*Certificate Authority*, CA).

**Tipos de certificados:**

- **DV (*Domain Validation*):** verifica solo la propiedad del dominio. Emision automatica, ideal para sitios web personales.
- **OV (*Organization Validation*):** verifica la identidad de la organizacion. Requiere verificacion manual.
- **EV (*Extended Validation*):** la validacion mas rigurosa. Muestra el nombre de la organizacion en la barra del navegador.

**Let's Encrypt** es una CA gratuita que automatiza la emision y renovacion de certificados DV, democratizando el HTTPS en la web.

```bash
# Certbot: gestionar certificados Let's Encrypt
sudo certbot --nginx -d ejemplo.com
sudo certbot renew
```

## Ataques comunes en la capa de aplicacion

**Man-in-the-Middle (MitM):** el atacante se interpone entre el cliente y el servidor, interceptando y potencialmente modificando la comunicacion. Se previene con TLS y la verificacion de certificados.

**Cross-Site Scripting (XSS):** el atacante inyecta scripts maliciosos en paginas web que otros usuarios visualizan. Se previene con sanitizacion de entrada y cabeceras Content-Security-Policy.

**SQL Injection:** el atacante inyecta consultas SQL en campos de entrada para acceder o manipular la base de datos. Se previene con consultas parametrizadas y ORM.

**Cross-Site Request Forgery (CSRF):** el atacante fuerza al navegador de un usuario autenticado a enviar peticiones no deseadas. Se previene con tokens CSRF.

## Cifrado de extremo a extremo

El cifrado de extremo a extremo (*end-to-end encryption*, E2EE) garantiza que solo el emisor y el receptor puedan leer los mensajes. Ni el servidor ni ningun intermediario pueden descifrar el contenido. Signal, WhatsApp (con Signal Protocol) y ProtonMail implementan E2EE.

```
Sin E2EE:  Cliente ──[TLS]── Servidor ──[TLS]── Destinatario
           (el servidor puede leer)

Con E2EE:  Cliente ──[E2EE]────────────────── Destinatario
           (el servidor NO puede leer)
```

> La seguridad en la capa de aplicacion es la ultima linea de defensa contra los ataques mas comunes. TLS protege los datos en transito; los certificados autentican la identidad; y las practicas de desarrollo seguro (OWASP Top 10) previenen las vulnerabilidades mas frecuentes. En un mundo donde la mayoria de las comunicaciones pasan por HTTPS, comprender la seguridad a este nivel es indispensable.

