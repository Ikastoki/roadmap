# Logging de red

El logging en aplicaciones de red no es un lujo sino una necesidad operativa. Sin registros adecuados, diagnosticar problemas de conectividad, timeouts o errores de protocolo equivale a buscar una aguja en un pajar. Java dispone de dos sistemas de logging principales: JUL (java.util.logging), incluido en el JDK, y SLF4J con Logback, que es el estandar de facto en la industria.

## JUL (java.util.logging)

JUL es el sistema de logging nativo de Java. Aunque menos flexible que SLF4J, es suficiente para aplicaciones pequenas y no requiere dependencias externas.

```java
import java.util.logging.*;

public class ConexionLogger {
    private static final Logger LOG =
            Logger.getLogger(ConexionLogger.class.getName());

    public void conectar(String host, int puerto) {
        LOG.fine("Intentando conexion a " + host + ":" + puerto);
        try {
            // conexion...
            LOG.info("Conectado a " + host + ":" + puerto);
        } catch (Exception e) {
            LOG.log(Level.SEVERE,
                    "Error al conectar a " + host, e);
        }
    }
}
```

### Niveles de log

JUL define siete niveles, de menor a mayor gravedad: `FINEST`, `FINER`, `FINE`, `CONFIG`, `INFO`, `WARNING`, `SEVERE`.

```java
LOG.finest("Traza de depuracion detallada");
LOG.finer("Traza de flujo");
LOG.fine("Depuracion");
LOG.info("Evento informativo");
LOG.warning("Advertencia");
LOG.severe("Error grave");
```

### Configuracion con archivo de propiedades

```properties
# logging.properties
handlers=java.util.logging.ConsoleHandler
.level=INFO
java.util.logging.ConsoleHandler.level=FINE
java.util.logging.ConsoleHandler.formatter=
    java.util.logging.SimpleFormatter
```

```bash
java -Djava.util.logging.config.file=logging.properties App
```

### Formato personalizado

```java
public class FormatoRed extends Formatter {
    @Override
    public String format(LogRecord record) {
        return String.format("[%1$tF %1$tT] [%2$s] %3$s%n",
                new java.util.Date(record.getMillis()),
                record.getLevel(),
                record.getMessage());
    }
}
```

## SLF4J + Logback

SLF4J es una fachada de logging que permite cambiar la implementacion subyacente (Logback, Log4j2, JUL) sin modificar el codigo. Logback es la implementacion mas utilizada por su rendimiento y flexibilidad.

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.16</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.5.12</version>
</dependency>
```

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ClienteHttp {
    private static final Logger log =
            LoggerFactory.getLogger(ClienteHttp.class);

    public void enviarPeticion(String url) {
        log.debug("Enviando peticion a {}", url);
        long inicio = System.currentTimeMillis();
        try {
            // ...
            long duracion = System.currentTimeMillis() - inicio;
            log.info("Peticion a {} completada en {}ms",
                    url, duracion);
        } catch (Exception e) {
            log.error("Error al ejecutar peticion a {}", url, e);
        }
    }
}
```

> SLF4J usa parametros con `{}`, lo que evita la concatenacion de strings cuando el nivel de log esta deshabilitado. Usar `log.debug("valor: " + x)` fuerza la concatenacion siempre; `log.debug("valor: {}", x)` solo la ejecuta si debug esta activo.

### Configuracion Logback (logback.xml)

```xml
<configuration>
    <appender name="CONSOLE"
              class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36}
                - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/red.log</file>
        <rollingPolicy
                class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/red.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{ISO8601} [%thread] %-5level %logger
                - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.ejemplo.red" level="DEBUG"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

## Logs de conexiones

Registrar cada conexion entrante y saliente es fundamental para auditoria y depuracion:

```java
log.info("Conexion entrante desde {}:{}",
        exchange.getRemoteAddress().getHostString(),
        exchange.getRemoteAddress().getPort());
log.info("Conexion saliente a {}:{} (timeout: {}ms)",
        host, puerto, timeout);
```

## Logs HTTP

Para depurar problemas con peticiones HTTP, es util registrar un resumen sin incluir cuerpos grandes:

```java
log.debug(">>> {} {} - Headers: {}",
        metodo, uri, cabeceras);
log.debug("<<< {} {} - {} bytes",
        codigoEstado, uri, tamanoRespuesta);
```

## No logear datos sensibles

Nunca deben registrarse contrasenas, tokens, API keys, datos personales o informacion financiera.

```java
// MAL: logea la cabecera Authorization completa
log.debug("Headers: {}", request.headers());

// BIEN: filtra cabeceras sensibles
Map<String, String> headersSeguros = new HashMap<>();
request.headers().forEach((k, v) -> {
    if ("authorization".equalsIgnoreCase(k)) {
        headersSeguros.put(k, "***");
    } else {
        headersSeguros.put(k, v);
    }
});
log.debug("Headers: {}", headersSeguros);
```

> Las herramientas de agregacion de logs (Splunk, ELK, Datadog) indexan todo lo que se les envie. Un token JWT registrado accidentalmente queda almacenado en el sistema de logs y puede ser accedido por personal no autorizado o comprometido si el sistema de logs es vulnerado.

## Buenas practicas

- Usar niveles de log adecuados: `TRACE` para datos crudos, `DEBUG` para flujo, `INFO` para eventos de negocio, `WARN` para situaciones recuperables, `ERROR` para fallos.
- Incluir identificadores de correlacion (request ID, session ID) en cada linea para rastrear el flujo completo.
- Rotar logs diariamente y retenerlos segun la politica de la organizacion.
- No bloquear el hilo principal con escritura de logs; Logback y Log4j2 escriben de forma asincrona con `AsyncAppender`.
