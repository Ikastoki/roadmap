# Autenticacion en MongoDB

MongoDB ofrece multiples mecanismos de autenticacion que permiten verificar la identidad de los usuarios y aplicaciones antes de conceder acceso a la base de datos. La eleccion del mecanismo depende del entorno de despliegue, los requerimientos de seguridad y la infraestructura existente. Cada mecanismo implica un flujo de credenciales distinto y un nivel de integracion diferente con sistemas corporativos.

## SCRAM (Salted Challenge Response Authentication Mechanism)

SCRAM es el mecanismo de autenticacion predeterminado en MongoDB desde la version 3.0. Implementa el protocolo definido en RFC 5802, utilizando un desafio criptografico que evita el envio de la contrasena en texto plano a traves de la red.

MongoDB emplea SCRAM-SHA-256 como algoritmo por defecto desde la version 4.0, aunque mantiene compatibilidad con SCRAM-SHA-1 para clientes heredados. Durante la autenticacion, el servidor genera un nonce (numero aleatorio de un solo uso) que el cliente debe combinar con su contrasena almacenada para producir una prueba criptografica. El servidor verifica esta prueba sin conocer la contrasena en texto plano.

La creacion de un usuario con autenticacion SCRAM se realiza mediante el metodo `db.createUser()`:

```javascript
use admin
db.createUser({
  user: "usuario_app",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "mis_datos" }],
  mechanisms: ["SCRAM-SHA-256"]
})
```

MongoDB almacena las credenciales SCRAM en la base de datos `admin`, dentro de la coleccion `system.users`, utilizando un hash con sal (salt) y un numero elevado de iteraciones (10.000 por defecto) para dificultar ataques de fuerza bruta.

> Aunque SCRAM-SHA-256 es el mecanismo por defecto, es posible habilitar exclusivamente este metodo desactivando SCRAM-SHA-1 en la configuracion del servidor mediante el parametro `authenticationMechanisms`.

## Autenticacion con certificados x.509

La autenticacion mediante certificados x.509 aprovecha la infraestructura de clave publica (PKI) para identificar clientes y servidores sin necesidad de contrasenas. Es el mecanismo recomendado en entornos donde ya existe una autoridad certificadora (CA) establecida.

MongoDB requiere que el servidor se ejecute con soporte TLS/SSL activado para utilizar este mecanismo. Cada cliente presenta un certificado firmado por la misma CA que firmo el certificado del servidor, o por una CA de confianza encadenada.

Para configurar la autenticacion x.509 es necesario:

1. Generar o adquirir certificados firmados por una CA de confianza.
2. Configurar el servidor MongoDB con TLS/SSL activado.
3. Crear un usuario cuyo `subject` del certificado coincida exactamente con el `DN` (Distinguished Name) del certificado del cliente.

```javascript
use admin
db.createUser({
  user: "CN=usuario,OU=Desarrollo,O=MiEmpresa,C=ES",
  roles: [{ role: "read", db: "reportes" }]
})
```

El cliente se autentica incluyendo su certificado y clave privada en la conexion:

```bash
mongosh --tls --tlsCertificateKeyFile cliente.pem \
        --tlsCAFile ca.pem \
        --authenticationDatabase admin \
        --authenticationMechanism MONGODB-X509 \
        --host servidor.ejemplo.com
```

> El `subject` del certificado debe coincidir exactamente con el valor almacenado en `system.users`. Una discrepancia, incluso en un espacio, provoca el rechazo de la autenticacion.

## Autenticacion contra LDAP

MongoDB Enterprise incluye la capacidad de delegar la autenticacion a un directorio LDAP (Lightweight Directory Access Protocol), permitiendo que los usuarios se autentiquen con las mismas credenciales que utilizan en el resto de la organizacion.

La integracion con LDAP requiere configurar el parametro `security.ldap` en el archivo de configuracion de MongoDB:

```yaml
security:
  ldap:
    servers: "ldap.corporacion.local"
    bind:
      method: "simple"
      queryUser: "cn=admin,dc=corporacion,dc=local"
      queryPassword: "contraseña_del_bind"
    transportSecurity: "tls"
    userToDNMatch:
      matchAttribute: "uid"
      baseDN: "ou=usuarios,dc=corporacion,dc=local"
```

MongoDB utiliza una cuenta de servicio (bind user) para consultar el directorio LDAP y verificar las credenciales del usuario que intenta conectarse. El servidor nunca almacena contraseñas LDAP localmente; actua unicamente como proxy de autenticacion.

Para crear un usuario que se autentique contra LDAP se utiliza:

```javascript
use admin
db.createUser({
  user: "jperez",
  roles: [{ role: "read", db: "ventas" }]
})
```

En este caso, la contrasena no se especifica en MongoDB porque se valida exclusivamente contra el directorio LDAP.

> LDAP solo esta disponible en MongoDB Enterprise. La comunidad debe utilizar SCRAM o x.509 como mecanismos de autenticacion.

## Autenticacion contra Kerberos

Kerberos es un protocolo de autenticacion basado en tickets que permite el inicio de sesion unico (SSO) en entornos Windows Active Directory o Linux con MIT Kerberos. MongoDB Enterprise puede integrarse como servicio Kerberos, validando los tickets que presentan los clientes.

La configuracion requiere definir un principal Kerberos para el servidor MongoDB y generar un keytab que contenga la clave del servicio:

```yaml
security:
  authenticationMechanisms: "GSSAPI"
setParameter:
  authenticationMechanisms: "GSSAPI"
```

El servidor MongoDB debe tener acceso al keytab y el archivo de configuracion `/etc/krb5.conf` debe estar correctamente definido con el realm y los KDC (Key Distribution Center).

Los clientes se autentican obteniendo un ticket del KDC y presentandolo al servidor MongoDB:

```bash
kinit jperez@CORPORACION.LOCAL
mongosh --host mongo.corporacion.local --authenticationMechanism GSSAPI
```

La creacion del usuario en MongoDB se realiza sin contrasena, ya que la autenticacion depende exclusivamente del ticket Kerberos:

```javascript
use admin
db.createUser({
  user: "jperez@CORPORACION.LOCAL",
  roles: [{ role: "readWrite", db: "finanzas" }]
})
```

> Kerberos depende de la sincronizacion de reloj entre todos los participantes (clientes, servidor MongoDB y KDC). Diferencias superiores a 5 minutos provocan el rechazo del ticket por considerarlo invalido.

