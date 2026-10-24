# Exception Handling Strategy

El manejo de excepciones en una aplicacion Spring Boot determina la experiencia del usuario cuando algo falla, la capacidad del equipo de operaciones para diagnosticar problemas y la seguridad del sistema al evitar la fuga de informacion interna. Una estrategia de excepciones bien definida centraliza el manejo de errores, define una jerarquia de excepciones consistente y produce respuestas HTTP estandarizadas que los clientes pueden procesar programaticamente.

## Handler global con @ControllerAdvice

`@ControllerAdvice` permite interceptar excepciones lanzadas desde cualquier `@Controller` o `@RestController` y transformarlas en respuestas HTTP antes de que lleguen al contenedor. Un handler global centraliza toda la logica de manejo de errores en un unico componente.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponseDTO> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        log.warn("Recurso no encontrado: {} en {}", ex.getMessage(), request.getRequestURI());

        ErrorResponseDTO error = ErrorResponseDTO.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .error("Recurso no encontrado")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponseDTO> handleBusiness(
            BusinessException ex, HttpServletRequest request) {
        log.error("Error de negocio: {}", ex.getMessage());

        ErrorResponseDTO error = ErrorResponseDTO.builder()
            .status(HttpStatus.UNPROCESSABLE_ENTITY.value())
            .error("Error de negocio")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponseDTO> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        Map<String, String> errores = ex.getBindingResult().getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "invalido",
                (a, b) -> a));

        ErrorResponseDTO error = ErrorResponseDTO.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Error de validacion")
            .message("La solicitud contiene errores de validacion")
            .details(errores)
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponseDTO> handleGeneric(
            Exception ex, HttpServletRequest request) {
        log.error("Error inesperado en {}: ", request.getRequestURI(), ex);

        ErrorResponseDTO error = ErrorResponseDTO.builder()
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Error interno del servidor")
            .message("Ha ocurrido un error inesperado")
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## Jerarquia de excepciones personalizadas

Una jerarquia de excepciones clara permite al codigo de negocio expresar intenciones semanticas y al handler global responder de forma especifica a cada tipo de error.

```java
// Excepcion base del dominio
public class DomainException extends RuntimeException {
    private final String codigo;

    public DomainException(String codigo, String mensaje) {
        super(mensaje);
        this.codigo = codigo;
    }

    public String getCodigo() {
        return codigo;
    }
}

// Excepciones especificas
public class ResourceNotFoundException extends DomainException {
    public ResourceNotFoundException(String recurso, Object id) {
        super("RECURSO_NO_ENCONTRADO",
              String.format("%s con id %s no encontrado", recurso, id));
    }
}

public class BusinessException extends DomainException {
    public BusinessException(String codigo, String mensaje) {
        super(codigo, mensaje);
    }
}

public class DuplicateResourceException extends DomainException {
    public DuplicateResourceException(String recurso, String campo, String valor) {
        super("RECURSO_DUPLICADO",
              String.format("%s con %s '%s' ya existe", recurso, campo, valor));
    }
}
```

```java
@ExceptionHandler(DuplicateResourceException.class)
public ResponseEntity<ErrorResponseDTO> handleDuplicate(
        DuplicateResourceException ex, HttpServletRequest request) {
    log.warn("Recurso duplicado: {}", ex.getMessage());

    ErrorResponseDTO error = ErrorResponseDTO.builder()
        .status(HttpStatus.CONFLICT.value())
        .error("Recurso duplicado")
        .message(ex.getMessage())
        .path(request.getRequestURI())
        .timestamp(Instant.now())
        .build();

    return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
}
```

> El handler generico para `Exception.class` debe ser el ultimo en la cadena y nunca exponer detalles internos del error al cliente. Los detalles completos se registran en el log del servidor para investigacion posterior.

## Cuerpo de respuesta estandarizado

Todos los errores devueltos por la API deben seguir una estructura consistente que los clientes puedan parsear de forma predecible.

```java
@Data
@Builder
public class ErrorResponseDTO {
    private int status;
    private String error;
    private String message;
    private Map<String, String> details;
    private String path;
    private Instant timestamp;
}
```<tool_call>
<function=write>
<parameter=content># Logging Strategy

Una estrategia de logging bien definida es fundamental para el diagnostico de problemas en produccion. El logging excesivo genera ruido que dificulta encontrar informacion relevante, mientras que el logging insuficiente deja gaps que impiden reconstruir la secuencia de eventos durante un incidente. La estrategia correcta balancea granularidad, rendimiento y utilidad operativa.

## SLF4J con Logback

Spring Boot utiliza SLF4J como fachada (facade) y Logback como implementacion por defecto. Esta separacion permite cambiar la implementacion de logging sin modificar el codigo de la aplicacion. SLF4J proporciona una API ligera y eficiente, mientras Logback ofrece un rendimiento optimizado y configuracion XML flexible.

```java
@Service
@Slf4j
public class PedidoService {

    public Pedido procesar(PedidoDTO dto) {
        log.debug("Procesando pedido: {}", dto);

        if (dto.getItems().isEmpty()) {
            log.warn("Pedido {} no tiene items", dto.getId());
            throw new BusinessException("PEDIDO_VACIO", "El pedido debe tener al menos un item");
        }

        try {
            Pedido pedido = repository.save(mapper.toEntity(dto));
            log.info("Pedido {} creado exitosamente con total {}",
                     pedido.getId(), pedido.getTotal());
            return pedido;
        } catch (DataAccessException e) {
            log.error("Error al persistir pedido {}: {}", dto.getId(), e.getMessage(), e);
            throw new PersistenceException("Error al guardar el pedido", e);
        }
    }
}
```

> Logback es significativamente mas rapido que Log4j 2 en escenarios de alta concurrencia porque utiliza buffers internos y operaciones de E/S no bloqueantes. Para la mayoria de aplicaciones Spring Boot, Logback es la eleccion correcta.

## Logging estructurado

El logging estructurado produce lineas de log en formato JSON que pueden ser parseadas automaticamente por herramientas de agregacion como ELK Stack, Loki o Datadog. Esto permite busquedas por campos, alertas basadas en patrones y correlacion de eventos.

```xml
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>requestUri</includeMdcKeyName>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <level>level</level>
                <logger>logger</logger>
                <message>message</message>
                <stackTrace>stacktrace</stackTrace>
            </fieldNames>
            <customFields>{"service":"mi-aplicacion","environment":"${ENVIRONMENT}"}</customFields>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON" />
    </root>
</configuration>
```

## Correlation IDs

Un correlation ID es un identificador unico que se genera en el primer endpoint de entrada y se propaga a traves de todas las llamadas internas y a servicios externos. Permite correlacionar todos los logs generados por una unica peticion a traves de multiples servicios y componentes.

```java
@Component
public class CorrelationFilter implements Filter {

    private static final String CORRELATION_HEADER = "X-Correlation-Id";
    private static final String CORRELATION_KEY = "correlationId";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String correlationId = httpRequest.getHeader(CORRELATION_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put(CORRELATION_KEY, correlationId);
        ((HttpServletResponse) response).setHeader(CORRELATION_HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_KEY);
        }
    }
}
```

```xml
<pattern>%d{ISO8601} [%thread] [%X{correlationId}] %-5level %logger{36} - %msg%n</pattern>
```

## MDC (Mapped Diagnostic Context)

MDC permite adjuntar datos contextuales a todos los logs generados desde el mismo hilo. Ademas del correlation ID, se pueden incluir el usuario autenticado, el nombre del endpoint o cualquier dato relevante para el diagnostico.

```java
@Component
public class RequestContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        MDC.put("method", httpRequest.getMethod());
        MDC.put("uri", httpRequest.getRequestURI());
        MDC.put("remoteAddr", httpRequest.getRemoteAddr());

        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            MDC.put("userId", auth.getName());
        }

        long startTime = System.currentTimeMillis();
        try {
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            MDC.put("duration", String.valueOf(duration));
            MDC.clear();
        }
    }
}
```

## Logs separados por dominio

Separar los logs de negocio de los logs de infraestructura permite aplicar politicas de retencion y nivel diferentes. Los logs de auditoria pueden retenerse 7 anos (requisito legal), mientras que los logs de debug se eliminan en 7 dias.

```xml
<!-- Logs de auditoria (negocio) -->
<appender name="AUDIT" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/mi-aplicacion/audit.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>/var/log/mi-aplicacion/audit.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>2555</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%d{ISO8601} [%X{correlationId}] [%X{userId}] %msg%n</pattern>
    </encoder>
</appender>

<!-- Logs de infraestructura -->
<appender name="INFRA" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/mi-aplicacion/infrastructure.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>/var/log/mi-aplicacion/infrastructure.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>30</maxHistory>
        <totalSizeCap>1GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
        <pattern>%d{ISO8601} [%thread] [%X{correlationId}] %-5level %logger{50} - %msg%n</pattern>
    </encoder>
</appender>

<!-- Niveles por paquete -->
<logger name="com.ejemplo.audit" level="INFO" additivity="false">
    <appender-ref ref="AUDIT" />
</logger>
<logger name="org.springframework" level="WARN" additivity="false">
    <appender-ref ref="INFRA" />
</logger>
<logger name="com.ejemplo" level="INFO">
    <appender-ref ref="INFRA" />
</logger>
```

