# Compresion, cifrado y codificaciones

La capa de presentacion gestiona las transformaciones que los datos sufren para optimizar su transmision y proteger su confidencialidad. La compresion reduce el tamano de los datos; el cifrado protege su contenido; y las codificaciones de caracteres garantizan que el texto sea legible entre sistemas diferentes.

Estas transformaciones ocurren transparentemente para el usuario: un navegador web descarga datos cifrados con TLS, los descifra, descomprime el contenido y renderiza el HTML en UTF-8, sin que el usuario perciba estas operaciones intermedias.

## Compresion de datos

La compresion es fundamental en las comunicaciones de red para reducir el volumen de datos transmitidos y el tiempo de transferencia.

**gzip y Brotli:** algoritmos de compresion de texto utilizados en HTTP. Los servidores web envian la respuesta comprimida (Content-Encoding: gzip o br), y el navegador la descomprime automaticamente. Brotli (desarrollado por Google) ofrece mejor compresion que gzip para contenido de texto web.

**Compresion de imagenes:** JPEG utiliza transformada coseno discreta (DCT) y cuantizacion para reducir el tamano eliminando informacion de alta frecuencia imperceptible al ojo humano. PNG utiliza compresion LZ77 sin perdida. WebP combina ambos enfoques.

**Compresion de video:** H.264/AVC y H.265/HEVC utilizan compresion inter-frame (diferencias entre frames consecutivos) y intra-frame (compresion dentro de un frame) para lograr ratios de compresion de 100:1 o mas con calidad perceptualmente buena.

## Cifrado de datos

### Cifrado simetrico

El cifrado simetrico utiliza una unica clave compartida entre emisor y receptor. Es rapido y eficiente, ideal para grandes volumenes de datos.

**AES (*Advanced Encryption Standard*):** estandar de cifrado simetrico con claves de 128, 192 o 256 bits. Es el cifrado mas utilizado en la actualidad para datos en reposo (disco duro cifrado, bases de datos) y en transito (TLS con AES-GCM).

**ChaCha20:** cifrado alternativo a AES, mas rapido en hardware sin aceleracion AES. Util en TLS (Google lo utiliza para conexiones Android) y en WireGuard (VPN moderna).

### Cifrado asimetrico

El cifrado asimetrico utiliza un par de claves: una publica (compartida con todos) y una privada (guardada por el propietario). La clave publica cifra y la privada descifra.

**RSA:** algoritmo basado en la dificultad de factorizar numeros grandes. Claves de 2048 o 4096 bits. Es el algoritmo asimetrico mas conocido pero se esta migrando a ECC por su mayor eficiencia.

**ECC (*Elliptic Curve Cryptography*):** algoritmo basado en curvas elipticas que ofrece la misma seguridad que RSA con claves mucho mas cortas (256 bits ECC ≈ 3072 bits RSA). Es mas rapido y consume menos recursos.

### TLS (*Transport Layer Security*)

TLS combina cifrado asimetrico (para el intercambio de claves) y simetrico (para la transmision de datos) en el protocolo que protege HTTPS, correo electronico y otras comunicaciones seguras:

```
1. ClientHello → ServerHello (negociacion de algoritmos)
2. Intercambio de claves (asimetrico)
3. Verificacion de certificado (CA)
4. Derivacion de clave simetrica
5. Transmision cifrada (simetrica)
```

TLS 1.3 (2018) redujo los roundtrips del handshake de 2 a 1, mejoro los algoritmos disponibles y elimino algoritmos obsoletos e inseguros.

## Codificaciones: ASCII, UTF-8, ASN.1

**ASCII** (*American Standard Code for Information Interchange*): codificacion de 7 bits que asigna un numero a 128 caracteres (32 control + 96 imprimibles). Es la base de todas las codificaciones modernas pero solo cubre caracteres basicos del ingles.

**UTF-8** (*Unicode Transformation Format*): codificacion variable que representa cualquier caracter del Unicode usando de 1 a 4 bytes. Los primeros 128 caracteres son compatibles con ASCII. Es la codificacion predominante en la web (98% de los sitios) y en sistemas Unix/Linux.

**ASN.1** (*Abstract Syntax Notation One*): notacion para definir estructuras de datos que pueden serializarse y deserializarse de forma unambiguous entre diferentes plataformas. Util en protocolos como SNMP, LDAP, X.509 (certificados) y kerberos.

> La capa de representacion es invisible pero omnipresente: cada pagina web que cargas, cada mensaje que envias y cada archivo que descargas pasa por compresion, cifrado y codificacion. Comprender estas transformaciones es esencial para diagnosticar problemas de compatibilidad (errores de codificacion), optimizar el rendimiento (eleccion de algoritmos de compresion) y garantizar la seguridad (configuracion de TLS).

