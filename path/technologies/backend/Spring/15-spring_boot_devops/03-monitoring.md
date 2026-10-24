# Monitoring

La monitorizacion de aplicaciones Spring Boot en produccion requiere una combinacion de recoleccion de metricas, agregacion de logs y visualizacion centralizada. Spring Boot Actuator proporciona los endpoints necesarios para exponer metricas y salud de la aplicacion, mientras que herramientas externas como Prometheus, Grafana y sistemas de logging centralizado completan el ecosistema de observabilidad.

## Spring Boot Actuator

Actuator es el componente fundamental que expone informacion operativa de la aplicacion. Su configuracion en produccion debe balancear la cantidad de informacion expuesta con la seguridad y el rendimiento.

```yaml
management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,info
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      group:
        readiness:
          include: db,redis
        liveness:
          include: diskspace,process
  metrics:
    export:
      prometheus:
        enabled: true
        step: 1m
    tags:
      application: mi-aplicacion
      environment: ${SPRING_PROFILES_ACTIVE:local}
```

Exponer Actuator en un puerto separado (8081) es una practica recomendada que permite restringir el acceso a los endpoints de monitorizacion sin afectar al trafico de la aplicacion principal.

## Prometheus

Prometheus es un sistema de recoleccion y almacenamiento de metricas basado en un modelo pull (el servidor Prometheus consulta periodicamente la aplicacion). La integracion con Spring Boot se realiza a traves de `micrometer-registry-prometheus`.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# Configuracion de Prometheus (prometheus.yml)
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8081']
        labels:
          environment: 'produccion'
```

```bash
# Verificar metricas disponibles en Prometheus format
curl http://localhost:8081/actuator/prometheus
```

Las metricas exportadas incluyen automaticamente `http_server_requests_seconds` (latencia y conteo de peticiones HTTP), `jvm_memory_used_bytes` (uso de memoria JVM), `hikaricp_connections_active` (conexiones activas del pool) y muchas otras metricas基础icas del ecosistema Micrometer.

## Grafana

Grafana proporciona dashboards visual para las metricas almacenadas en Prometheus. Spring Boot tiene dashboards pre-construidos que muestran informacion de JVM, HTTP, bases de datos y metricas de negocio.

```yaml
# docker-compose.yml para el stack de monitorizacion
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

> Los dashboards de Grafana para Spring Boot estan disponibles en la galeria publica de Grafana. Buscar "Spring Boot Statistics" o "Micrometer" para obtener dashboards pre-configurados que muestran latencia, trafico, errores y saturation (metodos RED).

## Logging centralizado

En un entorno con multiples instancias, los logs locales son insuficientes para el diagnostico de problemas. La agregacion centralizada de logs permite buscar y correlacionar eventos a traves de todas las instancias de la aplicacion.

```yaml
# Configuracion de Logback con appenders remotos
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/mi-aplicacion/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/mi-aplicacion/app.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{ISO8601} [%thread] %-5level %logger{36} - %X{correlationId} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="ELASTIC" class="ch.qos.logback.core.net.SocketAppender">
        <remoteHost>elasticsearch</remoteHost>
        <port>5044</port>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

El patagon RED (Rate, Errors, Duration) es un enfoque para monitorizar servicios que cuatro metricas fundamentales: tasa de peticiones, errores, latencia y trafico. Combinado con dashboards de Grafana y alertas de Prometheus, proporciona una visibilidad completa del estado de la aplicacion en produccion.

