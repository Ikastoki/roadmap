# Autenticacion y autorizacion en PostgreSQL

La seguridad en una base de datos no empieza con los permisos sobre las tablas, sino con la verificacion de quien se conecta y desde donde. PostgreSQL separa claramente dos conceptos que suelen confundirse: la autenticacion determina si un cliente puede establecer conexion, mientras que la autorizacion decide que operaciones puede realizar una vez conectado. Ambos mecanismos se complementan mediante el archivo `pg_hba.conf` y el sistema de privilegios basado en roles.

## Autenticacion

### pg_hba.conf

El archivo `pg_hba.conf` (_PostgreSQL Host-Based Authentication_) controla que clientes pueden conectarse a que bases de datos, con que nombre de usuario y mediante que metodo de autenticacion. Cada linea define una regla que se evalua de arriba a abajo; la primera que coincide con la conexion entrante determina el metodo de autenticacion:

```
# TYPE  DATABASE  USER  ADDRESS         METHOD
local   all       all                    peer
host    all       all    192.168.1.0/24  scram-sha-256
host    all       all    0.0.0.0/0       reject
```

`local` cubre conexiones mediante socket Unix, `host` conexiones TCP/IP. El campo `DATABASE` puede ser `all`, una base de datos especifica o `sameuser` para que el usuario solo pueda conectar a una base de datos con su mismo nombre. `ADDRESS` acepta CIDR o `0.0.0.0/0` para cualquier direccion.

> Las reglas se evaluan en orden. Es comun colocar las reglas mas restrictivas al principio y una regla `reject` al final como cortafuegos por defecto. La instruccion `pg_hba.conf` se recarga sin reiniciar con `pg_ctl reload` o `SELECT pg_reload_conf()`.

### Metodos de autenticacion

El metodo determina como se verifica la identidad del cliente. Cada uno ofrece un equilibrio distinto entre seguridad y conveniencia.

**Trust.** Concede acceso sin ninguna verificacion. Util solo en redes internas o entornos de desarrollo donde la seguridad se delega en el firewall:

```
local   all   all   trust
```

**Password.** Envia la contrasena en texto plano. Interceptable si no se usa SSL. Desaconsejado incluso en redes locales.

**MD5.** Aplica un hash MD5 a la contrasena antes de enviarla. Ofrece proteccion basica frente a sniffing, pero MD5 se considera debil desde hace anos. PostgreSQL 14 elimino el soporte nativo de MD5 como metodo de autenticacion; aun puede usarse si se configura explicitamente.

**SCRAM-SHA-256.** Metodo recomendado desde PostgreSQL 10 y obligatorio por defecto desde PostgreSQL 14. Implementa el protocolo SCRAM (_Salted Challenge Response Authentication Mechanism_) con SHA-256, que garantiza que la contrasena nunca viaja por la red ni siquiera hasheada de forma directa, sino que se demuestra el conocimiento de la misma mediante un reto criptografico:

```
host   all   all   0.0.0.0/0   scram-sha-256
```

Para generar contrasenas compatibles con SCRAM-SHA-256 se usa `CREATE ROLE ... PASSWORD` y PostgreSQL las almacena con el formato `SCRAM-SHA-256$<iterations>:<salt>$<stored_key>:<server_key>`.

**Cert.** Requiere que el cliente presente un certificado SSL firmado por una autoridad de confianza. La autenticacion se basa en el campo `CN` (Common Name) del certificado, que debe coincidir con el nombre del rol de PostgreSQL:

```
hostssl   all   all   0.0.0.0/0   cert   clientcert=verify-full
```

**LDAP.** Desvia la autenticacion a un servidor LDAP/Active Directory. El servidor LDAP valida las credenciales y PostgreSQL confia en el resultado. Se configura con los parametros `ldapserver`, `ldapprefix`, `ldapsuffix`:

```
host   all   all   10.0.0.0/8   ldap   ldapserver=ldap.example.com ldapprefix="uid=" ldapsuffix=",dc=example,dc=com"
```

**Kerberos (GSSAPI).** Utiliza tickets Kerberos para la autenticacion mutua entre cliente y servidor. Requiere que ambos tengan configurado el realm y el keytab correspondiente. Es el estandar en entornos corporativos con infraestructura de autenticacion centralizada.

### CREATE ROLE ... PASSWORD

La creacion de roles con contrasena almacena el hash en `pg_authid.rolpassword`. PostgreSQL no almacena la contrasena en texto plano, sino el resultado del algoritmo configurado (SCRAM-SHA-256 por defecto):

```sql
CREATE ROLE usuario_aplicacion WITH LOGIN PASSWORD 'pass_segura';
CREATE ROLE admin WITH LOGIN PASSWORD 'admin_2026' VALID UNTIL '2027-01-01';
ALTER ROLE usuario_aplicacion PASSWORD 'nueva_pass';
```

El atributo `VALID UNTIL` permite expirar roles temporalmente sin eliminarlos. Un rol expirado no puede iniciar sesion aunque la regla de `pg_hba.conf` lo permita.

## Autorizacion

### GRANT y REVOKE

Una vez autenticado, el rol debe tener permisos sobre los objetos que intenta usar. `GRANT` otorga privilegios y `REVOKE` los retira. La jerarquia de privilegios sigue la estructura de objetos:

```sql
GRANT CONNECT ON DATABASE mi_bd TO rol_lectura;
GRANT USAGE ON SCHEMA ventas TO rol_lectura;
GRANT SELECT ON ventas.productos TO rol_lectura;
```

Sin `CONNECT` sobre la base de datos no se puede establecer conexion. Sin `USAGE` sobre el esquema no se puede acceder a ningun objeto dentro de el, incluso teniendo permisos sobre la tabla. `GRANT` puede aplicarse a todas las tablas de un esquema de una vez:

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA ventas TO rol_lectura;
```

`REVOKE` sigue la misma sintaxis:

```sql
REVOKE INSERT, DELETE ON pedidos FROM rol_escritura;
REVOKE ALL PRIVILEGES ON SCHEMA ventas FROM PUBLIC;
```

La palabra clave `PUBLIC` representa a todos los roles, incluidos los futuros. Es una buena practica revocar privilegios de `PUBLIC` al crear una base de datos nueva.

### Privilegios

PostgreSQL define una docena de privilegios que se aplican segun el tipo de objeto:

| Privilegio | Objeto | Efecto |
|---|---|---|
| `SELECT` | tabla, vista, secuencia | Leer filas o valores |
| `INSERT` | tabla, vista | Insertar filas |
| `UPDATE` | tabla, vista | Modificar filas |
| `DELETE` | tabla, vista | Eliminar filas |
| `TRUNCATE` | tabla | Vaciar tabla |
| `REFERENCES` | tabla, columna | Crear foreign keys |
| `TRIGGER` | tabla | Crear triggers |
| `CREATE` | base de datos, esquema | Crear objetos dentro |
| `CONNECT` | base de datos | Conectar |
| `TEMPORARY` | base de datos | Crear tablas temporales |
| `EXECUTE` | funcion, procedimiento | Ejecutar |
| `USAGE` | esquema, secuencia, tipo | Acceder o usar |

Los privilegios sobre columnas permiten granularidad fina:

```sql
GRANT SELECT (id, nombre, email) ON clientes TO rol_soporte;
```

El propietario del objeto (el rol que lo creo) tiene todos los privilegios de forma implicita y no puede perderlos ni siquiera con `REVOKE`.

### DEFAULT PRIVILEGES

Cuando un usuario crea tablas en un esquema, los privilegios por defecto solo benefician al propietario y a `PUBLIC` segun el objeto. `ALTER DEFAULT PRIVILEGES` permite establecer privilegios que se aplicaran automaticamente a todos los objetos futuros de un tipo:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE dueno_bd IN SCHEMA ventas
    GRANT SELECT ON TABLES TO rol_lectura;

ALTER DEFAULT PRIVILEGES FOR ROLE dueno_bd IN SCHEMA ventas
    GRANT INSERT, UPDATE ON TABLES TO rol_escritura;
```

Los privilegios por defecto solo afectan a objetos creados por el rol especificado en `FOR ROLE`. Si `dueno_bd` crea una tabla nueva en `ventas`, `rol_lectura` obtendra automaticamente `SELECT` sobre ella. Las tablas existentes no se ven afectadas.

### pg_roles

La vista del catalogo `pg_roles` expone todos los roles del cluster con sus atributos:

```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin,
       rolvaliduntil, rolpassword
FROM pg_roles;
```

`rolpassword` no muestra la contrasena, solo indica si tiene contrasena asignada. La contrasena real se almacena en `pg_authid`, que solo es accesible para superusuarios.

### information_schema.role_table_grants

Para inspeccionar que permisos tiene cada rol sobre las tablas, el estandar SQL define la vista `information_schema.role_table_grants`:

```sql
SELECT grantor, grantee, table_schema, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'rol_lectura';
```

Esta vista muestra los privilegios otorgados explicitamente, no los heredados. Para ver la cadena completa de herencia es necesario consultar `pg_roles` y `pg_auth_members`:

```sql
SELECT r.rolname AS miembro,
       a.rolname AS pertenece_a
FROM pg_auth_members m
JOIN pg_roles r ON r.oid = m.member
JOIN pg_roles a ON a.oid = m.roleid;
```

