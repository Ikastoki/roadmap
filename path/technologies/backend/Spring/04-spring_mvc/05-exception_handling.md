# Exception Handling

El manejo de excepciones es una parte crítica del desarrollo de APIs REST, ya que las excepciones no manejadas resultan en respuestas de error genéricas que no proporcionan información útil al cliente. Spring MVC ofrece múltiples mecanismos para manejar excepciones de manera elegante, convirtiéndolas en respuestas HTTP significativas que siguen las mejores prácticas de diseño de APIs.

`@ExceptionHandler` es la anotación que define métodos para manejar excepciones específicas dentro de un controller. Cuando ocurre una excepción en un método del controller, Spring busca un método `@ExceptionHandler` que pueda manejar esa excepción específica o una de sus superclases. Esto permite un manejo granular de errores por controller.

```java
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioController {
    
    @GetMapping("/{id}")
    public Usuario obtener(@PathVariable Long id) {
        return usuarioService.obtenerPorId(id)
            .orElseThrow(() -> new ResourceNotFoundException("Usuario no encontrado: " + id));
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<Map<String, String>> manejarNoEncontrado(ResourceNotFoundException ex) {
        Map<String, String> error = new HashMap<>();
        error.put("mensaje", ex.getMessage());
        error.put("codigo", "RECURSO_NO_ENCONTRADO");
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> manejarValidacion(MethodArgumentNotValidException ex) {
        Map<String, Object> respuesta = new HashMap<>();
        List<String> errores = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());
        respuesta.put("errores", errores);
        return ResponseEntity.badRequest().body(respuesta);
    }
}
```

`@ControllerAdvice` y `@RestControllerAdvice` permiten definir manejo de excepciones global que se aplica a todos los controllers. Esto evita la duplicación de código de manejo de excepciones y centraliza la lógica de errores en un solo lugar. `@RestControllerAdvice` es la combinación de `@ControllerAdvice` y `@ResponseBody`, retornando respuestas serializadas directamente.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> manejarNoEncontrado(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            "Recurso no encontrado",
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> manejarNegocio(BusinessException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Error de negocio",
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> manejarGeneral(Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Error interno del servidor",
            "Ocurrió un error inesperado",
            LocalDateTime.now()
        );
        // Log del error para diagnóstico
        log.error("Error inesperado", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

`ResponseStatusException` es una excepción proporcionada por Spring que permite crear respuestas de error con código de estado HTTP específico. Esta excepción simplifica el manejo de errores al permitir especificar el estado HTTP, un mensaje y un reason directamente en la excepción, sin necesidad de métodos `@ExceptionHandler` adicionales.

```java
@Service
public class UsuarioService {
    
    public Usuario obtenerPorId(Long id) {
        return usuarioRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(
                HttpStatus.NOT_FOUND,
                "Usuario no encontrado con id: " + id
            ));
    }
    
    public Usuario crear(Usuario usuario) {
        if (usuarioRepository.existsByEmail(usuario.getEmail())) {
            throw new ResponseStatusException(
                HttpStatus.CONFLICT,
                "Ya existe un usuario con el email: " + usuario.getEmail()
            );
        }
        return usuarioRepository.save(usuario);
    }
}
```

El patrón de manejo de excepciones global típicamente incluye una clase base de excepciones personalizadas, un `@RestControllerAdvice` que maneja todas las excepciones, y clases de respuesta de error estandarizadas. Este patrón asegura que todas las respuestas de error sigan un formato consistente, facilitando el consumo por parte de los clientes de la API.

```java
// Excepción base personalizada
public class BusinessException extends RuntimeException {
    private final String codigo;
    
    public BusinessException(String codigo, String mensaje) {
        super(mensaje);
        this.codigo = codigo;
    }
    
    public String getCodigo() {
        return codigo;
    }
}

// Excepción específica
public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String recurso, Long id) {
        super("RECURSO_NO_ENCONTRADO", 
              String.format("%s no encontrado con id: %d", recurso, id));
    }
}

// Excepción de conflicto
public class ConflictException extends BusinessException {
    public ConflictException(String mensaje) {
        super("CONFLICTO", mensaje);
    }
}
```

Las excepciones personalizadas facilitan la creación de una jerarquía de errores clara y mantenible. Cada tipo de error de negocio se representa como una excepción específica, lo que permite un manejo diferenciado y mensajes de error más descriptivos. Esta aproximación también mejora la legibilidad del código, ya que las intenciones de negocio son evidentes en las excepciones lanzadas.

```java
@RestControllerAdvice
public class ExceptionHandlerConfig {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNoEncontrado(ResourceNotFoundException ex) {
        return buildResponse(HttpStatus.NOT_FOUND, ex.getCodigo(), ex.getMessage());
    }
    
    @ExceptionHandler(ConflictException.class)
    public ResponseEntity<ApiError> handleConflicto(ConflictException ex) {
        return buildResponse(HttpStatus.CONFLICT, ex.getCodigo(), ex.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidacion(MethodArgumentNotValidException ex) {
        List<String> detalles = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());
        
        ApiError error = new ApiError(
            HttpStatus.BAD_REQUEST.value(),
            "Error de validación",
            "VALIDACION_FALLIDA",
            detalles,
            LocalDateTime.now()
        );
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneral(Exception ex) {
        log.error("Excepción no manejada", ex);
        return buildResponse(HttpStatus.INTERNAL_SERVER_ERROR, 
            "ERROR_INTERNO", "Error interno del servidor");
    }
    
    private ResponseEntity<ApiError> buildResponse(HttpStatus status, 
                                                   String codigo, String mensaje) {
        ApiError error = new ApiError(
            status.value(), mensaje, codigo, null, LocalDateTime.now()
        );
        return ResponseEntity.status(status).body(error);
    }
}
```

El cuerpo de la respuesta de error (error response body) debe proporcionar información suficiente para que el cliente pueda entender y manejar el error. Un formato típico incluye el código de estado HTTP, un mensaje legible para humanos, un código de error específico de la aplicación, detalles adicionales y un timestamp. Este formato facilita el logging, el soporte técnico y el manejo programático de errores por parte de los clientes.

> `@ControllerAdvice` con `basePackages` permite restringir el alcance del manejo de excepciones a controllers en paquetes específicos. Esto es útil cuando se tiene una arquitectura modular con controllers en diferentes paquetes que requieren manejo de errores diferente.

