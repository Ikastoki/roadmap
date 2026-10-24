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
```

```java
@ExceptionHandler(DatosNoValidosException.class)
public ResponseEntity<ErrorResponseDTO> handleDatosNoValidos(
        DatosNoValidosException ex, HttpServletRequest request) {
    log.warn("Datos no validos: {}", ex.getMessage());

    ErrorResponseDTO error = ErrorResponseDTO.builder()
        .status(HttpStatus.BAD_REQUEST.value())
        .error("Datos no validos")
        .message(ex.getMessage())
        .details(ex.getCamposConError())
        .path(request.getRequestURI())
        .timestamp(Instant.now())
        .build();

    return ResponseEntity.badRequest().body(error);
}
```

## Logging de excepciones

El logging correcto de excepciones proporciona la informacion necesaria para diagnostico sin exponer datos sensibles al cliente. La clave esta en registrar el stack trace completo en el servidor mientras se devuelve un mensaje seguro al exterior.

```java
@ExceptionHandler(DataAccessException.class)
public ResponseEntity<ErrorResponseDTO> handleDataAccess(
        DataAccessException ex, HttpServletRequest request) {
    // Log completo para diagnostico interno
    log.error("Error de acceso a datos en {}: {}",
              request.getRequestURI(), ex.getMessage(), ex);

    // Mensaje seguro para el cliente
    ErrorResponseDTO error = ErrorResponseDTO.builder()
        .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
        .error("Error de persistencia")
        .message("Ha ocurrido un error al acceder a los datos")
        .path(request.getRequestURI())
        .timestamp(Instant.now())
        .build();

    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
}
```

## Excepciones de servicio externo

Las excepciones que provienen de llamadas a servicios externos requieren un manejo especial que incluya timeout, fallback y circuit breaker.

```java
@ExceptionHandler(ServiceUnavailableException.class)
public ResponseEntity<ErrorResponseDTO> handleServiceUnavailable(
        ServiceUnavailableException ex, HttpServletRequest request) {
    log.error("Servicio no disponible: {}", ex.getMessage());

    ErrorResponseDTO error = ErrorResponseDTO.builder()
        .status(HttpStatus.BAD_GATEWAY.value())
        .error("Servicio no disponible")
        .message("El servicio solicitado no esta disponible temporalmente")
        .path(request.getRequestURI())
        .timestamp(Instant.now())
        .build();

    return ResponseEntity.status(HttpStatus.BAD_GATEWAY).body(error);
}
```

```java
@ExceptionHandler(TimeoutException.class)
public ResponseEntity<ErrorResponseDTO> handleTimeout(
        TimeoutException ex, HttpServletRequest request) {
    log.warn("Timeout en {}: {}", request.getRequestURI(), ex.getMessage());

    ErrorResponseDTO error = ErrorResponseDTO.builder()
        .status(HttpStatus.GATEWAY_TIMEOUT.value())
        .error("Tiempo de espera agotado")
        .message("La operacion tardo mas de lo esperado")
        .path(request.getRequestURI())
        .timestamp(Instant.now())
        .build();

    return ResponseEntity.status(HttpStatus.GATEWAY_TIMEOUT).body(error);
}
```

