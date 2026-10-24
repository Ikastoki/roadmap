# Cifrado en MongoDB

La proteccion de los datos mediante cifrado opera en dos planos distintos: el cifrado en reposo protege los datos almacenados en el disco, mientras que el cifrado en transito protege los datos mientras viajan entre el cliente y el servidor o entre nodos del cluster. MongoDB ofrece mecanismos nativos para ambos escenarios, junto con integracion con sistemas externos de gestion de claves.

## Cifrado en reposo con WiredTiger

WiredTiger, el motor de almacenamiento predeterminado desde MongoDB 3.2, incorpora cifrado a nivel de pagina utilizando el estandar AES-256 en modo CBC (Cipher Block Chaining). Cada pagina de datos se cifra de forma independiente antes de escribirse en el disco y se descifra al ser cargada en memoria.

La activacion del cifrado en reposo requiere especificar la configuracion en el arranque del servidor:

```yaml
storage:
  wiredTiger:
    engineConfig:
      encryptWithKeyId: "clave_local"
      encryptionCipherMode: "AES256-CBC"
```

MongoDB soporta dos modalidades de gestion de claves: mediante un archivo local de clave (keyfile) o mediante un servidor KMIP (Key Management Interoperability Protocol).

### Cifrado con clave local

En esta modalidad, la clave de cifrado se almacena en un archivo accesible por el proceso `mongod`. El archivo debe contener una clave de 128, 192 o 256 bits codificada en base64. No existe proteccion adicional: quien acceda al archivo tiene acceso a todos los datos cifrados.

```yaml
security:
  enableEncryption: true
  encryptionKeyFile: "/etc/mongodb/clave-cifrado.key"
```

### Cifrado con KMIP

KMIP permite delegar la gestion de claves a un servidor externo especializado, eliminando la necesidad de almacenar claves localmente. MongoDB Enterprise soporta KMIP para la creacion, rotacion y almacenamiento seguro de claves de cifrado.

La configuracion requiere especificar la direccion del servidor KMIP y, opcionalmente, un certificado de cliente para la autenticacion mutua:

```yaml
security:
  kmip:
    serverName: "kmip.corporacion.local"
    port: 5696
    clientCertificateFile: "/etc/mongodb/kmip-cliente.pem"
    serverCAFile: "/etc/mongodb/kmip-ca.pem"
```

Cuando se utiliza KMIP, MongoDB solicita al servidor de claves la creacion de una clave maestra. A partir de esta, WiredTiger deriva claves hijas para cifrar cada archivo de datos individualmente.

> El cifrado en reposo de WiredTiger solo esta disponible en MongoDB Enterprise. La edicion Community no incluye esta funcionalidad y debe recurrir a soluciones de cifrado a nivel de sistema operativo (LUKS, dm-crypt) o de sistema de archivos.

## Cifrado en transito con TLS/SSL

El cifrado en transito protege las comunicaciones entre los clientes y el servidor MongoDB, asi como el trafico entre nodos de un replica set o cluster fragmentado. MongoDB utiliza TLS (Transport Layer Security) como protocolo de cifrado, con soporte para las versiones 1.2 y 1.3.

### Configuracion del servidor

Para habilitar TLS en el servidor MongoDB es necesario proporcionar un certificado firmado por una autoridad certificadora (CA) y su clave privada asociada. La configuracion se realiza mediante los parametros `net.tls` o los equivalentes heredados `net.ssl`:

```yaml
net:
  tls:
    mode: "requireTLS"
    certificateKeyFile: "/etc/mongodb/servidor.pem"
    CAFile: "/etc/mongodb/ca.pem"
    allowConnectionsWithoutCertificates: false
```

El parametro `mode` admite los siguientes valores:

- **disabled**: TLS desactivado. Es el valor por defecto y no se recomienda en entornos productivos.
- **allowTLS**: el servidor acepta conexiones tanto TLS como sin cifrar. Util para migraciones graduales.
- **preferTLS**: similar a `allowTLS`, pero el servidor anuncia preferencia por conexiones TLS.
- **requireTLS**: solo se aceptan conexiones TLS. Las conexiones sin cifrar son rechazadas.

### Conexion desde el cliente

Los clientes deben especificar los parametros TLS al conectar:

```bash
mongosh --tls --tlsCAFile ca.pem \
        --host servidor.ejemplo.com \
        --port 27017
```

Si el servidor requiere autenticacion mutua (mutual TLS), el cliente debe presentar su propio certificado:

```bash
mongosh --tls --tlsCertificateKeyFile cliente.pem \
        --tlsCAFile ca.pem \
        --tlsAllowInvalidCertificates
```

> El uso de `--tlsAllowInvalidCertificates` esta pensado exclusivamente para entornos de desarrollo. En produccion debe verificarse siempre la cadena de confianza del certificado del servidor.

### Certificados

Los certificados deben cumplir los siguientes requisitos:

- Estar en formato PEM, combinando el certificado y la clave privada en un unico archivo (o en archivos separados mediante `tlsCertificateKeyFile` y `tlsPrivateKeyFile`).
- Incluir el `subjectAltName` (SAN) correspondiente al nombre del servidor. MongoDB verifica que el nombre del host coincida con el SAN del certificado.
- Tener una cadena de confianza completa hasta una CA raiz de confianza.

Para generar un certificado autofirmado valido para pruebas:

```bash
openssl req -x509 -newkey rsa:4096 -days 365 -nodes \
  -keyout servidor.key -out servidor.crt \
  -subj "/CN=servidor.ejemplo.com" \
  -addext "subjectAltName=DNS:servidor.ejemplo.com"
cat servidor.crt servidor.key > servidor.pem
```

### Cifrado entre nodos del replica set

En un replica set, la comunicacion interna entre nodos (replicacion, heartbeats, elecciones) tambien puede cifrarse mediante TLS. La configuracion es la misma que para las conexiones de cliente, pero aplicada al trafico interno:

```yaml
net:
  tls:
    mode: "requireTLS"
    clusterFile: "/etc/mongodb/cluster.pem"
    clusterCAFile: "/etc/mongodb/ca.pem"
```

El parametro `clusterFile` especifica un certificado distinto para la comunicacion entre miembros del cluster, aunque puede reutilizarse el mismo certificado del servidor si es necesario.

> El rendimiento se ve afectado por el cifrado TLS. En cargas de trabajo intensivas se recomienda evaluar el impacto y considerar la aceleracion por hardware (ASI/SIMD) o la terminacion TLS en un balanceador de carga.

