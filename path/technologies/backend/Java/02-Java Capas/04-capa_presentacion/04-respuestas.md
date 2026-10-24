# Formateo de respuestas

Una API que devuelve respuestas inconsistentes (distinta estructura para cada endpoint, codigos de estado arbitrarios, mensajes de error en formato variable) genera clientes fragiles y dificulta la depuracion. El formateo estandarizado de respuestas garantiza que cualquier cliente pueda interpretar el resultado de una peticion sin necesidad de conocer los detalles internos de cada operacion.

La capa de presentacion es la responsable de construir y devolver estas respuestas. El Service nunca debe formatear una respuesta HTTP: su responsabilidad termina al devolver un resultado (o lanzar una excepcion). Es el Controller quien decide como se traduce ese resultado al lenguaje de la capa de transporte.

## Respuestas estandarizadas (ApiResponse)

Una respuesta estandarizada envuelve el resultado de la operacion en una estructura predecible que incluye al menos tres campos: un indicador de exito, un mensaje descriptivo y los datos devueltos. Opcionalmente, puede incluir metadatos adicionales como la marca de tiempo o un codigo de error interno.

```java
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private LocalDateTime timestamp;

    public ApiResponse(boolean success, String message, T data) {
        this.success = success;
        this.message = message;
        this.data = data;
        this.timestamp = LocalDateTime.now();
    }

    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, "Operacion exitosa", data);
    }

    public static <T> ApiResponse<T> created(T data) {
        return new ApiResponse<>(true, "Recurso creado exitosamente", data);
    }

    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, message, null);
    }

    // Getters...
}
```

El uso de genericos (`ApiResponse<T>`) permite que el campo `data` adopte el tipo adecuado en cada respuesta, manteniendo la estructura envolvente constante. El cliente sabe que siempre recibira un objeto JSON con las claves `success`, `message`, `data` y `timestamp`, independientemente del endpoint al que llame.

```json
{
    "success": true,
    "message": "Operacion exitosa",
    "data": {
        "id": 1,
        "name": "Juan Perez",
        "email": "juan@example.com"
    },
    "timestamp": "2026-07-28T10:30:00"
}
```

## Codigos de estado HTTP

Cada operacion debe devolver el codigo de estado HTTP que refleje con precision el resultado de la peticion. Usar siempre 200 OK para todas las respuestas exitosas y 400 Bad Request para todos los errores priva al cliente de informacion valiosa sobre la naturaleza del resultado.

Los codigos mas relevantes en una API REST construida con arquitectura de capas son:

| Codigo | Significado | Cuando usarlo |
|---|---|---|
| **200 OK** | La peticion se proceso correctamente | GET exitsoso, PUT/PATCH exitoso |
| **201 Created** | Se creo un nuevo recurso | POST que crea un recurso |
| **204 No Content** | La peticion se proceso pero no hay contenido que devolver | DELETE exitoso, actualizacion sin cuerpo |
| **400 Bad Request** | La peticion esta malformada | Error de validacion, parametros invalidos |
| **401 Unauthorized** | El cliente no esta autenticado | Token faltante o invalido |
| **403 Forbidden** | El cliente no tiene permisos | Acceso denegado a un recurso |
| **404 Not Found** | El recurso solicitado no existe | ID de recurso inexistente |
| **409 Conflict** | Conflicto con el estado actual del recurso | Violacion de unicidad, estado inconsistente |
| **422 Unprocessable Entity** | La peticion es semanticamente incorrecta | Reglas de negocio incumplidas |
| **500 Internal Server Error** | Error inesperado del servidor | Excepcion no controlada, fallo de infraestructura |

```java
public Response getUser(Long id) {
    try {
        UserDTO user = userService.findById(id);
        return Response.ok(ApiResponse.ok(user)).build();
    } catch (NotFoundException e) {
        return Response.status(404, ApiResponse.error(e.getMessage())).build();
    }
}
```

> La diferencia entre 400 y 422 es sutil pero importante: 400 indica que la peticion no pudo ser parseada o validada a nivel de sintaxis; 422 indica que la peticion es sintacticamente valida pero semanticamente incorrecta (p. ej., un email que ya existe en el sistema).

## Manejo de errores consistente

Cuando ocurre un error, la respuesta debe mantener la misma estructura que una respuesta exitosa, pero con `success: false` y el campo `data` en `null`. El mensaje de error debe ser descriptivo sin revelar detalles internos del sistema (stack traces, nombres de tablas, versiones de librerias).

```json
{
    "success": false,
    "message": "Usuario no encontrado con ID: 42",
    "data": null,
    "timestamp": "2026-07-28T10:30:00"
}
```

Para errores de validacion, es util incluir detalles adicionales que permitan al cliente identificar exactamente que campo fallo y por que:

```java
public class ValidationErrorResponse {
    private boolean success;
    private String message;
    private List<FieldError> errors;
    private LocalDateTime timestamp;

    public static class FieldError {
        private String field;
        private String rejectedValue;
        private String reason;
    }
}
```

```json
{
    "success": false,
    "message": "Error de validacion",
    "errors": [
        {
            "field": "email",
            "rejectedValue": "invalido",
            "reason": "Formato de email invalido"
        },
        {
            "field": "age",
            "rejectedValue": -1,
            "reason": "Debe ser un numero positivo"
        }
    ],
    "timestamp": "2026-07-28T10:30:00"
}
```

El manejo centralizado de errores mediante un handler global (como `@ControllerAdvice` en Spring o un `ExceptionMapper` en Jakarta REST) captura todas las excepciones lanzadas por los controladores y las traduce a la respuesta estandarizada. Esto elimina la necesidad de bloques try-catch en cada metodo del Controller y garantiza que ningun error escape sin ser formateado correctamente.

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public Response handleNotFound(NotFoundException e) {
        return Response.status(404, ApiResponse.error(e.getMessage())).build();
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Response handleValidation(MethodArgumentNotValidException e) {
        List<FieldError> errors = e.getBindingResult().getFieldErrors().stream()
            .map(error -> new FieldError(error.getField(), error.getRejectedValue(), error.getDefaultMessage()))
            .toList();
        return Response.status(400, new ValidationErrorResponse(errors)).build();
    }

    @ExceptionHandler(Exception.class)
    public Response handleGeneral(Exception e) {
        return Response.status(500, ApiResponse.error("Error interno del servidor")).build();
    }
}
```

Un manejo de errores bien implementado convierte las excepciones tecnicas (NullPointerException, DataIntegrityViolationException, ConstraintViolationException) en mensajes de error comprensibles para el cliente, sin filtrar informacion interna y manteniendo la estructura de respuesta consistente en toda la API.
