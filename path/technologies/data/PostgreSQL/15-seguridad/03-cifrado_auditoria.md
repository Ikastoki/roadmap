# Cifrado y auditoria

La seguridad no termina en el control de acceso a las filas. Los datos almacenados y en transito pueden quedar expuestos si el disco es robado, el backup se filtra o alguien intercepta la conexion de red. El cifrado protege la confidencialidad de los datos tanto en reposo como en movimiento, mientras que la auditoria genera un registro inmutable de las operaciones ejecutadas, esencial para cumplimiento normativo y deteccion de incidentes.

## Cifrado con pgcrypto

La extension `pgcrypto` proporciona funciones criptograficas dentro del motor de PostgreSQL, permitiendo cifrar datos antes de almacenarlos y descifrarlos al leerlos. No es una alternativa a SSL ni al cifrado de disco; opera a nivel de columna para datos que deben permanecer ilegibles incluso para administradores de base de datos.

```sql
CREATE EXTENSION pgcrypto;
```

### Cifrado simetrico con PGP

`pgp_sym_encrypt` y `pgp_sym_decrypt` utilizan un algoritmo simetrico (AES-128 por defecto, configurable a AES-256) con una sola contrasena compartida entre emisor y receptor:

```sql
-- cifrar
INSERT INTO tarjetas (numero_cifrado)
VALUES (pgp_sym_encrypt('4111111111111111', 'clave_secreta'));

-- descifrar
SELECT pgp_sym_decrypt(numero_cifrado, 'clave_secreta')
FROM tarjetas
WHERE id = 1;
```

El texto cifrado incluye metadatos como un hash SHA-1 del texto plano, que permite verificar integridad al descifrar. Si la contrasena es incorrecta, `pgp_sym_decrypt` lanza un error.

> La contrasena viaja en la consulta SQL y puede quedar registrada en logs si `log_statement` esta activo. Es recomendable pasar la clave mediante `current_setting()` o variables de sesion en lugar de literales.

### Cifrado asimetrico con PGP

`pgp_pub_encrypt` y `pgp_pub_decrypt` utilizan un par de claves publica/privada. Cualquier rol puede cifrar con la clave publica, pero solo quien posee la clave privada puede descifrar:

```sql
-- generar par de claves (fuera de PostgreSQL con gpg)
-- Importar clave publica y privada al servidor

-- cifrar con clave publica
INSERT INTO documentos (contenido_cifrado)
VALUES (pgp_pub_encrypt('contenido sensible', dearmor('-----BEGIN PGP PUBLIC KEY BLOCK----- ...')));

-- descifrar con clave privada
SELECT pgp_pub_decrypt(contenido_cifrado, dearmor('-----BEGIN PGP PRIVATE KEY BLOCK----- ...'), 'passphrase')
FROM documentos
WHERE id = 1;
```

El cifrado asimetrico es util cuando multiples fuentes deben poder escribir datos cifrados pero solo un destinatario central debe poder leerlos.

## SSL/TLS

El cifrado en transito protege los datos mientras viajan entre el cliente y el servidor. PostgreSQL soporta SSL/TLS nativamente mediante la configuracion de `postgresql.conf` y archivos de certificado.

### Configuracion del servidor

Para habilitar SSL, se establece `ssl = on` y se proporcionan los certificados:

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'       # opcional, para verificar clientes
ssl_crl_file = '/etc/ssl/certs/ca.crl'       # opcional, lista de revocacion
ssl_min_protocol_version = 'TLSv1.3'         # minimo TLS 1.3 recomendado
```

PostgreSQL soporta TLS 1.2 y 1.3. Se recomienda desactivar versiones anteriores y cifrados debiles mediante `ssl_ciphers`:

```ini
ssl_ciphers = 'HIGH:!aNULL:!eNULL:!MD5'
```

### Conexiones SSL

En `pg_hba.conf`, el tipo `hostssl` exige SSL para la conexion:

```
hostssl   all   all   0.0.0.0/0   scram-sha-256
```

Si el cliente intenta conectar sin SSL y la regla es `hostssl`, la conexion se rechaza. El tipo `host` permite tanto SSL como conexiones sin cifrar.

Para exigir verificacion del certificado del cliente:

```
hostssl   all   all   0.0.0.0/0   cert   clientcert=verify-full
```

### Verificacion desde el cliente

Los clientes se conectan con `sslmode` que determina el nivel de verificacion:

| sslmode | Comportamiento |
|---|---|
| `disable` | No usa SSL |
| `allow` | Intenta SSL, cae a sin cifrar |
| `prefer` | SSL si el servidor lo soporta |
| `require` | SSL obligatorio pero sin verificar CA |
| `verify-ca` | SSL + verificar certificado servidor contra CA |
| `verify-full` | SSL + verificar CA + verificar nombre del servidor |

El modo `verify-full` evita ataques man-in-the-middle al comprobar que el CN o SAN del certificado coincide con el hostname de conexion.

## Auditoria

La auditoria en PostgreSQL puede abordarse desde dos niveles: el log nativo del servidor y extensiones especializadas como `pgaudit`.

### log_statement

El parametro `log_statement` controla que sentencias SQL se registran en el log del servidor. Sus valores posibles son:

- `none` (por defecto): no se registra ninguna sentencia.
- `ddl`: solo sentencias de definicion de datos (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `COMMENT`, `GRANT`, `REVOKE`).
- `mod`: todas las sentencias DDL y DML que modifican datos (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `COPY FROM`).
- `all`: todas las sentencias, incluyendo `SELECT`.

```ini
log_statement = 'mod'
```

> `log_statement` registra el texto de la sentencia antes de ejecutarla, por lo que no permite saber si la operacion tuvo exito o fallo. Para eso se necesita `log_min_error_statement`.

### log_connections y log_disconnections

Estos parametros registran cada conexion y desconexion del servidor, incluyendo direccion IP, usuario y duracion de la sesion:

```ini
log_connections = on
log_disconnections = on
```

Cada entrada de conexion incluye la linea de `pg_hba.conf` que la autorizo, lo que facilita el diagnostico de reglas de autenticacion. Las desconexiones muestran el tiempo total de sesion, util para detectar conexiones que nunca se cierran.

### pgaudit

`pgaudit` es una extension que proporciona auditoria detallada a nivel de objeto y de sesion, muy superior al log nativo. Se instala como extension:

```sql
CREATE EXTENSION pgaudit;
```

Se configura en `postgresql.conf`:

```ini
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write,ddl'
pgaudit.log_level = 'notice'
pgaudit.log_catalog = off
pgaudit.log_relation = on
pgaudit.log_statement_once = off
```

El parametro `pgaudit.log` acepta una lista de clases de operaciones:

| Clase | Operaciones auditadas |
|---|---|
| `read` | SELECT, COPY TO |
| `write` | INSERT, UPDATE, DELETE, TRUNCATE, COPY FROM |
| `function` | Llamadas a funciones y procedimientos |
| `role` | GRANT, REVOKE, CREATE/ALTER/DROP ROLE |
| `ddl` | Todas las sentencias DDL |
| `misc` | Comandos variados (DISCARD, FETCH, CHECKPOINT) |
| `all` | Todas las anteriores |

Para auditoria por objeto, `pgaudit` permite definir politicas especificas que solo registran acceso a tablas concretas:

```sql
CREATE EXTENSION pgaudit;

-- auditar solo accesos a tabla facturas
SELECT pgaudit.audit_table('facturas');
```

Las entradas de `pgaudit` aparecen en los logs con el prefijo `AUDIT:`, facilitando su extraccion con herramientas como pgBadger o scripts personalizados.

### pg_stat_statements

Aunque no es una herramienta de auditoria en sentido estricto, `pg_stat_statements` registra estadisticas de ejecucion de todas las sentencias SQL: numero de ejecuciones, tiempo total, tiempo de CPU, filas afectadas, lecturas de bloques, etc. Es util para detectar consultas anomalas o comportamientos sospechosos:

```sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

`pg_stat_statements` normaliza las consultas reemplazando valores literales por parametros (`$1`, `$2`), lo que evita exponer datos sensibles en los logs al tiempo que permite agrupar consultas identicas en estructura.

