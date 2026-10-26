# Formatos de datos

La capa de presentacion se encarga de la representacion, formato e intercambio de datos entre sistemas que pueden utilizar codificaciones y formatos diferentes. Su funcion es asegurar que la informacion enviada por un sistema sea legible y comprensible por el receptor, independientemente de las diferencias en la arquitectura de hardware o software.

En TCP/IP, las funciones de la capa de presentacion suelen ser asumidas por las aplicaciones. Sin embargo, el concepto es fundamental para comprender como se representan y transforman los datos en las comunicaciones de red.

## Representacion de datos

Los datos pueden representarse en multiples formatos segun su naturaleza:

**Texto:** codificado como secuencias de bytes segun una tabla de caracteres (ASCII, UTF-8, ISO-8859-1). El texto plano es la representacion mas basica; el HTML, XML y JSON son formatos de texto estructurado.

**Imagenes:** representadas como maps de bits (BMP, JPEG, PNG, GIF, TIFF) o como formatos vectoriales (SVG). Cada formato utiliza algoritmos de compresion diferentes que afectan la calidad y el tamano del archivo.

**Audio:** representado como muestras digitales de la onda sonora. Formatos sin compresion (WAV, AIFF) y con compresion (MP3, AAC, FLAC, OGG). La calidad depende de la frecuencia de muestreo (44.1 kHz, 48 kHz) y la profundidad de bits (16, 24 bits).

**Video:** secuencia de imagenes (frames) combinadas con audio. Formatos de contenedor (MP4, MKV, WebM, AVI) que pueden contener multiples codecs de video (H.264, H.265, VP9, AV1) y audio (AAC, MP3, Opus).

## Formatos comunes de intercambio

**JSON (*JavaScript Object Notation*):** formato de texto ligero y legible por humanos, ampliamente utilizado en APIs web y configuraciones. Estructura basada en pares de clave-valor y arrays.

```json
{
  "nombre": "servidor-web-01",
  "ip": "10.0.1.50",
  "puertos": [80, 443],
  "activo": true
}
```

**XML (*Extensible Markup Language*):** formato de texto mas verbose que JSON pero con capacidades avanzadas como esquemas de validacion (XSD), transformaciones (XSLT) y namespaces. Comun en SOAP, RSS y documentos ofimaticos (OOXML, ODF).

**Protocol Buffers (protobuf):** formato binario eficiente utilizado por gRPC y otras APIs de alto rendimiento. Mas compacto que JSON pero no legible por humanos.

**YAML (*YAML Ain't Markup Language*):** formato legible por humanos comun en configuraciones (Docker, Kubernetes, Ansible). Soporta anidamiento complejo y tipos de datos豐富os.

## Compresion y cifrado

### Compresion

La compresion reduce el tamano de los datos para optimizar el uso del ancho de banda y el almacenamiento:

**Compresion sin perdida:** preserva todos los datos originales. ZIP, gzip, brotli (texto), PNG (imagenes), FLAC (audio). Ideal para datos donde la integridad es critica.

**Compresion con perdida:** elimina informacion percibida como redundante o imperceptible para reducir drasticamente el tamano. JPEG (imagenes), MP3/AAC (audio), H.264/H.265 (video). La calidad se controla con un factor de compresion.

### Cifrado

El cifrado protege los datos contra acceso no autorizado:

**Simetrico (AES, ChaCha20):** la misma clave se usa para cifrar y descifrar. Rapido pero requiere intercambio seguro de claves.

**Asimetrico (RSA, ECC):** un par de claves (publica/privada). La clave publica cifra y la privada descifra. Mas lento pero resuelve el problema del intercambio de claves.

**Hashing (SHA-256, SHA-3):** funcion unidireccional que genera un resumen de los datos. Util para verificacion de integridad, no para descifrado.

## Codificaciones de caracteres

**ASCII:** codificacion de 7 bits que representa 128 caracteres (letras, numeros, simbolos basicos). Base de todas las codificaciones occidentales pero insuficiente para caracteres internacionales.

**UTF-8:** codificacion variable de 1 a 4 bytes que puede representar todos los caracteres del Unicode. Es la codificacion predominante en la web y en Linux. Compatible con ASCII para los primeros 128 caracteres.

**UTF-16:** codificacion de 2 o 4 bytes. Util en Windows internamente y en Java. Menos eficiente que UTF-8 para texto occidental.

**Problemas de compatibilidad:** cuando un sistema envia texto en UTF-8 y el receptor espera ISO-8859-1, los caracteres con tildes y eñes aparecen como caracteres erroneos. La deteccion automatica de codificacion y el uso consistente de UTF-8 previenen estos problemas.

> La capa de representacion es donde los bits se convierten en informacion significativa para los humanos y las aplicaciones. La eleccion de formatos, codificaciones y algoritmos de compresion/cifrado tiene un impacto directo en la interoperabilidad, el rendimiento y la seguridad de las comunicaciones de red.

