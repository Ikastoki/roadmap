# Transacciones distribuidas (JTA)

Cuando una operacion debe modificar datos en multiples recursos transaccionales (varias bases de datos, colas de mensajes, sistemas de archivos) de forma atomica, las transacciones locales no son suficientes. Se necesitan transacciones distribuidas gestionadas mediante Java Transaction API (JTA) y el protocolo two-phase commit (2PC).

## Java Transaction API (JTA)

JTA es una especificacion Java que define una interfaz estandar para coordinar transacciones que abarcan multiples recursos. Forma parte de Jakarta EE (antes Java EE), aunque puede utilizarse en aplicaciones standalone con implementaciones como Atomikos o Narayana.

Los componentes principales de JTA son:
- **UserTransaction**: interfaz para que la aplicacion controle los limites de la transaccion.
- **TransactionManager**: gestiona el ciclo de vida de las transacciones a nivel del contenedor.
- **XAResource**: interfaz que deben implementar los recursos (bases de datos, JMS) para participar en transacciones distribuidas.

```java
// Uso directo de UserTransaction en un entorno Jakarta EE
@Resource
private UserTransaction userTransaction;

public void operacionDistribuida() {
    try {
        userTransaction.begin();

        // Operacion en base de datos A
        dsA.getConnection().createStatement()
            .execute("INSERT INTO tabla_a VALUES (1)");

        // Operacion en base de datos B
        dsB.getConnection().createStatement()
            .execute("INSERT INTO tabla_b VALUES (2)");

        // Envio de mensaje a cola JMS
        jmsTemplate.convertAndSend("cola.pedidos", mensaje);

        userTransaction.commit();
    } catch (Exception e) {
        userTransaction.rollback();
        throw e;
    }
}
```

## Transacciones XA

XA es un estandar definido por The Open Group para coordinar transacciones distribuidas. Un recurso XA (como una base de datos o un broker de mensajeria) implementa la interfaz `XAResource` y participa en el protocolo 2PC.

```java
// Configuracion de un DataSource XA con Atomikos
@Bean(initMethod = "init", destroyMethod = "close")
public AtomikosDataSourceBean dataSourceA() {
    AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
    ds.setUniqueResourceName("XADB_A");
    ds.setXaDataSourceClassName("org.postgresql.xa.PGXADataSource");
    ds.setXaProperties(xaProperties("jdbc:postgresql://hostA:5432/dbA"));
    return ds;
}

@Bean(initMethod = "init", destroyMethod = "close")
public AtomikosDataSourceBean dataSourceB() {
    AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
    ds.setUniqueResourceName("XADB_B");
    ds.setXaDataSourceClassName("org.postgresql.xa.PGXADataSource");
    ds.setXaProperties(xaProperties("jdbc:postgresql://hostB:5432/dbB"));
    return ds;
}

private Properties xaProperties(String url) {
    Properties props = new Properties();
    props.setProperty("url", url);
    props.setProperty("user", "usuario");
    props.setProperty("password", "contrasena");
    return props;
}
```

Cada recurso XA debe tener un nombre unico (`uniqueResourceName`) para que el gestor transaccional pueda identificarlo en el log de transacciones.

## Two-Phase Commit (2PC)

El protocolo 2PC garantiza que todos los recursos implicados en una transaccion distribuida se confirmen o se reviertan atomicamente. Consta de dos fases:

### Fase 1: Preparacion (votacion)

El coordinador de transacciones (TransactionManager) pregunta a cada recurso XA si esta listo para confirmar.

```
Coordinador                   Recurso A                  Recurso B
    |                              |                          |
    |---- prepare() ------------>|                          |
    |                              |---- prepare() -------->|
    |                              |                          |
    |<--- vote YES ---------------|                          |
    |                              |<------- vote YES -------|
```

Cada recurso escribe en su log de transacciones la informacion necesaria para poder confirmar o revertir, y responde YES o NO.

### Fase 2: Confirmacion (commit/rollback)

Si todos los recursos votaron YES, el coordinador ordena el commit. Si alguno vota NO o falla, el coordinador ordena el rollback en todos los recursos.

```
Coordinador                   Recurso A                  Recurso B
    |                              |                          |
    |---- commit() ------------->|                          |
    |                              |---- commit() --------->|
    |                              |                          |
    |<--- committed ---------------|                          |
    |                              |<-------- committed -----|
```

> El punto de no retorno en 2PC es tras la fase de preparacion. Si un recurso falla durante la fase de commit, el coordinador debe reintentar el commit hasta que tenga exito. Esto puede dejar la transaccion en un estado indeterminado durante un tiempo, que solo se resuelve cuando el recurso se recupera.

## UserTransaction vs container-managed

### UserTransaction (gestion programatica)

La aplicacion controla explicitamente los limites de la transaccion mediante la interfaz `UserTransaction`. Es util cuando la logica de negocio requiere control fino sobre el momento de inicio y fin de la transaccion.

```java
@Resource
private UserTransaction userTransaction;

public void operacion() {
    userTransaction.begin();
    try {
        // Logica de negocio
        userTransaction.commit();
    } catch (Exception e) {
        userTransaction.setRollbackOnly();
        userTransaction.rollback();
    }
}
```

### Container-managed (gestion declarativa)

El contenedor (Jakarta EE o Spring con JTA) gestiona automaticamente los limites transaccionales mediante anotaciones o XML.

```java
// Jakarta EE
@TransactionAttribute(TransactionAttributeType.REQUIRED)
public void operacion() {
    // El contenedor inicia y finaliza la transaccion
}

// Spring con JTA
@Transactional
@Transactional("jtaTransactionManager")
public void operacion() {
    // Spring gestiona la transaccion JTA
}
```

El enfoque declarativo reduce el codigo boilerplate y delega la gestion de recursos al contenedor, que puede optimizar el uso de conexiones y la recuperacion ante fallos.

## Atomikos

Atomikos Essentials es una implementacion popular de JTA para aplicaciones standalone y Spring Boot. Proporciona un gestor transaccional ligero que implementa 2PC sin necesidad de un servidor de aplicaciones Jakarta EE.

```xml
<dependency>
    <groupId>com.atomikos</groupId>
    <artifactId>transactions-spring-boot3-starter</artifactId>
    <version>6.0.0</version>
</dependency>
```

```properties
# Configuracion Atomikos en Spring Boot
spring.jta.atomikos.properties.max-timeout=300000
spring.jta.atomikos.properties.default-jta-timeout=30000
spring.jta.atomikos.properties.allow-sub-pools=true
spring.jta.atomikos.properties.log-base-dir=./tx-logs
spring.jta.enabled=true
```

## Narayana

Narayana es la implementacion de JTA desarrollada por Red Hat, utilizada en WildFly y JBoss. Es una de las implementaciones mas maduras y completas, con soporte para recuperacion automatica y logs de transacciones.

```xml
<dependency>
    <groupId>org.jboss.narayana.jta</groupId>
    <artifactId>narayana-jta</artifactId>
    <version>7.0.0.Final</version>
</dependency>
```

## Consideraciones de rendimiento

Las transacciones distribuidas tienen un coste significativo:
- Mayor latencia: cada operacion requiere mensajes de ida y vuelta entre el coordinador y los recursos.
- Bloqueo prolongado: los recursos mantienen bloqueos durante ambas fases del 2PC.
- Punto unico de fallo: el coordinador de transacciones es un punto critico.

> Antes de adoptar JTA, hay que evaluar si realmente se necesita atomicidad entre recursos. En muchos casos, el diseno puede simplificarse usando el patron SAGA (transacciones compensatorias) o replicacion asincrona de datos, evitando la complejidad y el coste de 2PC.

```text
Alternativas a JTA:
- Patron SAGA: transacciones locales encadenadas con compensaciones.
- Replicacion asincrona: eventual consistency entre bases de datos.
- Servicio unico: consolidar los datos en una sola base de datos.
- Eventos y CQRS: separar escrituras y lecturas.
```
