# Logging

Spring Boot utiliza Logback como sistema de logging por defecto, proporcionando una configuración automática que funciona sin necesidad de configuración adicional. Logback es el sucesor de Log4j y ofrece mejor rendimiento y flexibilidad, siendo la implementación recomendada para aplicaciones Spring Boot.

La configuración de logging se puede realizar mediante propiedades en `application.properties` o `application.yml`. Spring Boot ofrece propiedades específicas para configurar niveles de logging, patrones de salida, archivos de log y rotación de archivos. Esta configuración es suficiente para la mayoría de casos de uso sin necesidad de configuración XML personalizada.

```properties
# Nivel de logging global
logging.level.root=INFO

# Nivel para paquetes específicos
logging.level.com.miaplicacion=DEBUG
logging.level.org.springframework=WARNING
logging.level.org.hibernate=ERROR

# Archivo de log
logging.file.name=logs/mi-aplicacion.log
logging.file.max-size=10MB
logging.file.max-history=30

# Patrón de log
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
```

```yaml
# Configuración equivalente en YAML
logging:
  level:
    root: INFO
    com.miaplicacion: DEBUG
    org.springframework: WARNING
  file:
    name: logs/mi-aplicacion.log
    max-size: 10MB
    max-history: 30
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

Los niveles de logging disponibles son TRACE, DEBUG, INFO, WARNING, ERROR y FATAL. Cada nivel incluye los niveles superiores, por lo que configurar un nivel en WARNING mostrará mensajes de WARNING, ERROR y FATAL. La configuración por paquetes permite establecer diferentes niveles para diferentes partes de la aplicación, facilitando el diagnóstico de problemas específicos.

Log4j2 se puede usar como alternativa a Logback mediante la exclusión del starter de logging por defecto y la adición del starter de Log4j2. Esto es útil en entornos que requieren características avanzadas de Log4j2 como layouts personalizados, filters y appenderers específicos.

```xml
<!-- Excluir Logback y agregar Log4j2 en pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

La configuración personalizada de logging se puede realizar mediante archivos XML como `logback-spring.xml` o `logback.xml` ubicados en `src/main/resources`. El archivo `logback-spring.xml` es preferido porque soporta perfiles de Spring Boot, permitiendo configuraciones diferentes según el entorno.

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>
    
    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/app.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/app.%d{yyyy-MM-dd}.log</fileNamePattern>
                <maxHistory>30</maxHistory>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="FILE" />
        </root>
    </springProfile>
</configuration>
```

> La configuración de logging en producción debe balancear la granularidad necesaria para diagnóstico con el rendimiento y el espacio en disco. Un nivel demasiado verboso puede impactar el rendimiento mientras que uno demasiado restrictivo puede dificultar la resolución de problemas.

