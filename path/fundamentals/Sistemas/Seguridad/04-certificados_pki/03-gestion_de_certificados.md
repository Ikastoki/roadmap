# Gestion de Certificados

La obtencion de certificados digitales validos fue durante mucho tiempo un proceso costoso y burocratico que requeria verificacion manual de identidad y pago de cuotas anuales significativas. La aparicion de Let's Encrypt y el protocolo ACME revoluciono este panorama al democratizar el acceso a certificados TLS validos y gratuitos, acelerando significativamente la adopcion de HTTPS en internet. Gestionar correctamente el ciclo de vida de los certificados es esencial para mantener la disponibilidad y seguridad de cualquier servicio que dependa de comunicaciones TLS.

## Let's Encrypt y ACME

Let's Encrypt es una autoridad de certificacion gratuita, automatizada y abierta fundada en 2015 por Internet Security Research Group (ISRG). Su mision es reducir las barreras economicas para la adopcion de HTTPS, proporcionando certificados TLS validos sin costo. Utiliza el protocolo ACME (Automatic Certificate Management Environment) para automatizar completamente el proceso de validacion, emision y renovacion de certificados.

El protocolo ACME define un conjunto de interacciones automatizadas entre el cliente (certbot u otro agente) y el servidor de Let's Encrypt:

- **Challenge-Response**: para demostrar control sobre un dominio, Let's Encrypt ofrece multiples tipos de validacion. HTTP-01 requiere colocar un archivo de validacion en el directorio `.well-known/acme-challenge/` del servidor web. DNS-01 requiere crear un registro TXT especifico en la zona DNS del dominio, siendo ideal para certificados wildcard. TLS-ALPN-01 valida mediante una conexion TLS especial.
- **Emision**: una vez validado el control del dominio, el cliente genera un par de claves y envia una solicitud de certificado firmada con su clave privada
- **Renovacion**: los certificados de Let's Encrypt tienen una validez de 90 dias, lo que incentiva la automatizacion de renovaciones

Certbot es el cliente ACME mas utilizado, aunque existen alternativas como acme.sh, Caddy (con ACME integrado) y Traefik. La instalacion basica con certbot para un servidor Apache seria:

```bash
certbot --apache -d ejemplo.com -d www.ejemplo.com
```

> Let's Encrypt no emite certificados Organization Validation (OV) o Extended Validation (EV). Solo emite certificados Domain Validation (DV), lo que significa que verifica control sobre el dominio pero no la identidad juridica de la organizacion. Para certificados OV o EV, se requieren CA comerciales como DigiCert, Sectigo o GlobalSign.

## Expiracion y Renovacion Automatica

La expiracion de certificados es una de las causas mas comunes de interrupciones de servicio no planificadas en produccion. Un certificado expirado provoca errores en el navegador que impiden el acceso a servicios web, API y aplicaciones moviles. La renovacion automatica es la defensa fundamental contra este escenario, eliminando la dependencia de procesos manuales propensos a errores humanos.

La estrategia de renovacion automatica debe considerar varios factores:

- **Ventana de renovacion**: renovar con suficiente antelacion para manejar fallos de validacion o problemas de red. Un periodo tipico es renovar cuando quedan 30 dias o menos de validez
- **Notificaciones**: configurar alertas antes de la expiracion como measure de seguridad adicional. Herramientas como `certbot certificates` muestran el estado actual de los certificados
- **Verificacion post-renovacion**: despues de renovar, verificar que el nuevo certificado esta correctamente instalado y sirviendo
- **Logs de auditoria**: registrar todas las operaciones de renovacion para investigacion de problemas

En Kubernetes, cert-manager automatiza la gestion de certificados mediante Custom Resources. Monitoriza certificados TLS y renueva automaticamente cuando se acercan a la expiracion, integrandose nativamente con Let's Encrypt y otras CA. La configuracion tipica incluye ClusterIssuer, Certificate y Secret que almacena el certificado y la clave privada.

## Certificados Auto-firmados

Un certificado auto-firmado es un certificado X.509 firmado por la misma entidad cuya identidad certifica, en lugar de por una CA externa. Esto significa que no tiene cadena de confianza hacia una CA raiz conocida y los navegadores mostraran advertencias de seguridad al acceder a servicios que lo utilicen. Sin embargo, los certificados auto-firmados tienen usos legitimos en escenarios donde la confianza se establece por otros medios.

Los principales usos de certificados auto-firmados incluyen:

- **Entornos de desarrollo y testing**: donde la validacion de CA no es necesaria y se busca simplicidad
- **Servicios internos**: comunicaciones entre microservicios dentro de una red privada donde la confianza se establece por politica de red
- **Infraestructura de clave privada**: CI/CD pipelines, bases de datos y sistemas que requieren cifrado interno sin exposicion publica
- **Prototipos y Proof of Concept**: rapida implementacion sin dependencias externas

Para generar un certificado auto-firmado con OpenSSL:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout clave_privada.pem -out certificado.pem \
  -subj "/CN=ejemplo.com/O=Mi Organizacion"
```

> Aunque los certificados auto-firmados son funcionales, no deben utilizarse en servicios expuestos a internet. Los navegadores no confian en ellos por defecto, lo que genera experiencias de usuario negativas y potenciales vulnerabilidades si los usuarios aprenden a ignorar las advertencias de seguridad.

## mTLS (Mutual TLS)

El TLS estandar solo valida la identidad del servidor hacia el cliente: el servidor presenta su certificado y el cliente verifica que es legitimo. En mTLS, ambas partes intercambian certificados y validan mutuamente sus identidades. Esto crea un canal bidireccionalmente autenticado donde tanto el cliente como el servidor deben poseer certificados validos emitidos por una CA de confianza compartida.

mTLS es especialmente relevante en arquitecturas de microservicios donde la zero-trust security es un requisito. En estos entornos, no se asume confianza por pertenencia a una red. Cada servicio debe autenticarse ante cada otro servicio al que accede, independientemente de su ubicacion de red. Service mesh como Istio y Linkerd implementan mTLS automaticamente entre los sidecar proxies, transparente para las aplicaciones.

El flujo de mTLS funciona de la siguiente manera:

1. El cliente inicia la conexion TLS y recibe el certificado del servidor
2. El servidor solicita el certificado del cliente (CertificateRequest)
3. El cliente envia su certificado y demuestra posesion de la clave privada
4. Ambas partes validan los certificados recibidos contra su almacen de confianza
5. La conexion se establece solo si ambas validaciones son exitosas

La gestion de certificados en entornos mTLS a gran escala requiere automatizacion significativa. Cada servicio necesita su propio certificado con un ciclo de vida corto (tipicamente horas o dias), lo que hace impracticable la gestion manual. Herramientas como Vault de HashiCorp, cert-manager en Kubernetes y SPIFFE/SPIRE proporcionan la infraestructura necesaria para emitir y rotar certificados de forma automatizada y segura.

---

