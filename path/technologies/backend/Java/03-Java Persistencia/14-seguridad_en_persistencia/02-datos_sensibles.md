# Datos sensibles

Almacenar datos sensibles (contrasenas, numeros de tarjeta, datos personales identificables) en la base de datos implica riesgos legales y tecnicos. Regulaciones como el RGPD en Europa o la PCI DSS en el sector financiero exigen que ciertos datos se almacenen cifrados, enmascarados o al menos hashados. JPA y Hibernate proporcionan mecanismos para aplicar transformaciones a nivel de columna y entidad sin que la capa de negocio tenga que preocuparse por la criptografia.

## @ColumnTransformer

La anotacion `@ColumnTransformer` de Hibernate permite especificar expresiones SQL personalizadas para leer y escribir una columna. Es util para aplicar funciones de cifrado o hashing directamente en la base de datos sin modificar el modelo.

```java
@Entity
@Table(name = "tarjetas")
public class TarjetaCredito {

    @Id
    private Long id;

    @Column(name = "numero")
    @ColumnTransformer(
        read = "pgp_sym_decrypt(numero, 'clave-secreta')",
        write = "pgp_sym_encrypt(?, 'clave-secreta')"
    )
    private String numeroTarjeta;

    @Column(name = "cvv")
    @ColumnTransformer(
        read = "pgp_sym_decrypt(cvv, 'clave-secreta')",
        write = "pgp_sym_encrypt(?, 'clave-secreta')"
    )
    private String cvv;

    @Column(name = "fecha_caducidad")
    @ColumnTransformer(
        read = "pgp_sym_decrypt(fecha_caducidad, 'clave-secreta')",
        write = "pgp_sym_encrypt(?, 'clave-secreta')"
    )
    private String fechaCaducidad;

    public boolean ultimosCuatroDigitosValidos(String ultimosCuatro) {
        return numeroTarjeta != null && numeroTarjeta.endsWith(ultimosCuatro);
    }
}
```

> `@ColumnTransformer` delega el cifrado en la base de datos, lo que implica que la clave debe gestionarse a nivel de servidor de BD. La expresion `write` usa `?` como placeholder del valor Java, no del JDBC; Hibernate lo vincula automaticamente.

Las funciones `pgp_sym_encrypt`/`pgp_sym_decrypt` son especificas de PostgreSQL con el extension `pgcrypto`. Para otros motores hay que usar funciones equivalentes (AES_ENCRYPT en MySQL, `ENCRYPT` en SQL Server).

## AttributeConverter

`AttributeConverter` es el mecanismo estandar de JPA para transformar un atributo de la entidad en un valor de columna y viceversa. Es la alternativa portable a `@ColumnTransformer` y permite aplicar logica arbitraria (cifrado, hashing, serializacion) antes de la persistencia.

```java
@Converter
public class PasswordConverter implements AttributeConverter<String, String> {

    private static final String ALGORITHM = "BCrypt";

    @Override
    public String convertToDatabaseColumn(String passwordPlano) {
        if (passwordPlano == null) return null;
        // BCrypt genera el hash automaticamente con salt incorporado
        return BCrypt.withDefaults().hashToString(12, passwordPlano.toCharArray());
    }

    @Override
    public String convertToEntityAttribute(String passwordHash) {
        // Nunca se debe devolver el hash al exterior; no se descifra
        return passwordHash;
    }
}
```

Aplicacion en la entidad:

```java
@Entity
@Table(name = "usuarios")
public class Usuario {

    @Id
    private Long id;

    @Column(name = "email")
    private String email;

    @Column(name = "password")
    @Convert(converter = PasswordConverter.class)
    private String password;

    public boolean passwordCoincide(String passwordPlano) {
        return BCrypt.verifyer().verify(passwordPlano.toCharArray(), password);
    }
}
```

El converter se puede aplicar a nivel de campo o de clase, y puede ser utilizado globalmente registrandolo en el `persistence.xml` o via `@Converter(autoApply = true)`.

## JPA EncryptedType

Hibernate ofrece un tipo personalizado llamado `EncryptedType` que simplifica el cifrado simetrico de columnas. Se configura declarativamente combinando un `StringType` con un cipher JCE.

```java
@Entity
@Table(name = "datos_sensibles")
public class DatoSensible {

    @Id
    private Long id;

    @Type(
        value = EncryptedType.class,
        parameters = {
            @Parameter(name = "algorithm", value = "AES/GCM/NoPadding"),
            @Parameter(name = "password", value = "clave-32-caracteres-exactos"),
        }
    )
    private String numeroSeguridadSocial;

    @Column(name = "descripcion")
    private String descripcion;
}
```

> El tipo `EncryptedType` esta disponible en Hibernate 5.x y superiores mediante el modulo `hibernate-jpamodelgen`. La clave debe tener la longitud exacta que exige el algoritmo AES (16, 24 o 32 bytes segun la variante).

## Transparent Data Encryption (TDE)

TDE es una caracteristica del motor de base de datos que cifra los ficheros de datos en reposo (datafiles, logs, backups) de forma transparente para la aplicacion. La base de datos descifra automaticamente los datos cuando se leen y los cifra al escribirlos, sin que la aplicacion necesite modificaciones.

- **PostgreSQL**: no tiene TDE nativo; se usan extensiones como `pg_tde` o cifrado a nivel de filesystem (LUKS, dm-crypt).
- **MySQL**: soporta TDE mediante InnoDB tablespace encryption con la variable `innodb_encrypt_tables`.
- **SQL Server**: TDE integrado desde 2008, gestionado con `CREATE DATABASE ENCRYPTION KEY`.
- **Oracle**: TDE integrado, soporta cifrado de columnas y tablespaces completos.

TDE complementa pero no sustituye al cifrado a nivel de aplicacion. Mientras que TDE protege contra accesos fisicos a los discos (robo de backups, administradores de sistema), el cifrado a nivel de JPA protege contra accesos logicos (administradores de BD que ejecutan consultas directas).

## Nunca logear contrasenas

No importa que mecanismo de cifrado se utilice si la aplicacion registra las contrasenas en texto plano en los logs. Una configuracion erronea de logging puede exponer datos sensibles en archivos de log accesibles por personal operativo o sistemas de monitoreo.

```java
// MAL: logea la contrasena
log.info("Creando usuario con email {} y password {}", email, password);

// BIEN: nunca incluir datos sensibles
log.info("Creando usuario con email {}", email);

// BIEN: enmascarar si es necesario para depuracion
log.info("Creando usuario con email {} y password [PROTEGIDO]", email);
```

Spring Boot proporciona `LoggingEventPattern` con conversores personalizados para filtrar campos sensibles, y librerias como Logback permiten definir `PatternLayout` con reemplazos regex. Ademas, la configuracion de `hibernate.show_sql` y el logging de `org.hibernate.type.descriptor.sql.BasicBinder` pueden exponer valores en las consultas: en produccion estos logs deben estar desactivados o configurados para enmascarar ciertos campos.

> Las contrasenas nunca deben almacenarse cifradas simetricamente (reversibles), sino hasheadas con algoritmos de derivacion de clave como BCrypt, SCrypt o Argon2. Si se pierde la clave de cifrado simetrico, todos los datos quedan expuestos.
