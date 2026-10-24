# Excepciones de negocio

Las excepciones de negocio son clases de excepcion personalizadas que representan situaciones especificas del dominio: un email duplicado, un saldo insuficiente, un producto sin stock, un pedido en un estado que no permite cierta operacion. Al modelar estas situaciones como excepciones propias, la capa de negocio puede comunicar de forma explicita y tipada que ha ocurrido una condicion anomala que la capa de presentacion debe manejar.

## Excepciones personalizadas

Una excepcion de negocio debe extender `RuntimeException` (unchecked) en la mayoria de los casos, aunque existen argumentos para usar excepciones checked en determinados escenarios. La eleccion determina si el codigo cliente esta obligado a declarar o capturar la excepcion.

```java
public abstract class BusinessException extends RuntimeException {
    private final String code;
    private final String description;

    protected BusinessException(String code, String message) {
        super(message);
        this.code = code;
        this.description = message;
    }

    protected BusinessException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
        this.description = message;
    }

    public String getCode() {
        return code;
    }

    public String getDescription() {
        return description;
    }
}
```

A partir de esta clase base, se crean excepciones especificas para cada situacion de negocio:

```java
public class DuplicateEmailException extends BusinessException {
    public DuplicateEmailException(Email email) {
        super("EMAIL_DUPLICATE",
              "El email " + email.value() + " ya esta registrado en el sistema");
    }
}

public class InsufficientBalanceException extends BusinessException {
    public InsufficientBalanceException(Money balance, Money required) {
        super("INSUFFICIENT_BALANCE",
              "Saldo insuficiente. Disponible: " + balance + ", requerido: " + required);
    }
}

public class OrderStatusTransitionException extends BusinessException {
    public OrderStatusTransitionException(OrderId id, OrderStatus current, OrderStatus target) {
        super("INVALID_STATUS_TRANSITION",
              "No se puede cambiar el pedido " + id + " de " + current + " a " + target);
    }
}
```

Cada excepcion incluye un codigo de error unico (`EMAIL_DUPLICATE`, `INSUFFICIENT_BALANCE`) que la capa de presentacion puede usar para construir respuestas estandarizadas sin depender del texto del mensaje. Esto es especialmente util para aplicaciones internacionalizadas o APIs publicas.

## Checked vs Unchecked

La eleccion entre excepciones checked y unchecked en la capa de negocio es una decision arquitectonica con implicaciones en el diseno del codigo.

Las excepciones checked (`extends Exception`) obligan al invocador a declararlas o capturarlas. Esto puede ser apropiado cuando se quiere forzar al llamante a considerar explicitamente un escenario de error. Sin embargo, en la practica, las excepciones checked tienden a propagarse a traves de todas las capas, ensuciando las firmas de los metodos y creando cadenas de `throws` que nada aportan.

```java
// Checked: obliga a declarar o capturar en cada capa
public class PaymentService {
    public PaymentResult processPayment(Order order) throws PaymentFailedException {
        // ...
    }
}

public class OrderService {
    public Order createOrder(CreateOrderRequest request) throws PaymentFailedException {
        // ...
    }
}

public class OrderController {
    public Response createOrder(CreateOrderRequest request) {
        try {
            Order order = orderService.createOrder(request);
            return Response.ok(order);
        } catch (PaymentFailedException e) {
            return Response.badRequest(e.getMessage());
        }
    }
}
```

Las excepciones unchecked (`extends RuntimeException`) no obligan a declararlas ni capturarlas. El invocador puede ignorarlas o capturarlas en un punto centralizado. Esta es la opcion mas utilizada en la capa de negocio porque permite que las excepciones fluyan hacia un handler global sin contaminar las firmas de los metodos.

> La regla general: usa excepciones unchecked para errores de negocio, donde el invocador no puede hacer nada para recuperarse en el momento (salvo informar al usuario). Usa excepciones checked cuando el invocador deba tomar una accion correctiva obligatoriamente (por ejemplo, un archivo no encontrado donde el usuario deba elegir otro).

## Manejo centralizado de errores

Las excepciones de negocio no deben capturarse en cada Service ni en cada metodo del Controller. En lugar de eso, se manejan en un punto unico que se encarga de interpretar la excepcion, registrarla y construir la respuesta adecuada.

```java
public class GlobalExceptionHandler {
    private final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    public ApiResponse handle(Throwable throwable) {
        if (throwable instanceof BusinessException ex) {
            logger.warn("Error de negocio: {} - {}", ex.getCode(), ex.getDescription());
            return ApiResponse.error(ex.getCode(), ex.getDescription(), 400);
        }
        if (throwable instanceof EntityNotFoundException ex) {
            logger.warn("Entidad no encontrada: {}", ex.getMessage());
            return ApiResponse.error("NOT_FOUND", ex.getMessage(), 404);
        }
        if (throwable instanceof IllegalArgumentException ex) {
            logger.warn("Argumento invalido: {}", ex.getMessage());
            return ApiResponse.error("VALIDATION_ERROR", ex.getMessage(), 400);
        }

        logger.error("Error inesperado", throwable);
        return ApiResponse.error("INTERNAL_ERROR", "Ha ocurrido un error interno", 500);
    }
}
```

El handler global se coloca tipicamente en la capa de presentacion y se registra como el punto de entrada para todas las excepciones no capturadas. En Spring, esto se logra con `@ControllerAdvice` y `@ExceptionHandler`. En Java puro, puede ser un filtro en el punto de entrada de la aplicacion (un middleware en un servidor HTTP, un interceptor en un framework).

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DuplicateEmailException.class)
    public ResponseEntity<ApiError> handleDuplicateEmail(DuplicateEmailException ex) {
        ApiError error = new ApiError("EMAIL_DUPLICATE", ex.getMessage(), HttpStatus.CONFLICT.value());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(InsufficientBalanceException.class)
    public ResponseEntity<ApiError> handleInsufficientBalance(InsufficientBalanceException ex) {
        ApiError error = new ApiError("INSUFFICIENT_BALANCE", ex.getMessage(), HttpStatus.BAD_REQUEST.value());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiError> handleBusinessException(BusinessException ex) {
        ApiError error = new ApiError(ex.getCode(), ex.getDescription(), HttpStatus.BAD_REQUEST.value());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
}
```

## Mapeo a HTTP

Cada tipo de excepcion de negocio se mapea a un codigo de estado HTTP especifico. Este mapeo es responsabilidad del handler global y debe ser consistente en toda la aplicacion.

| Excepcion | Codigo HTTP | Significado |
|-----------|-------------|-------------|
| `DuplicateEmailException` | 409 Conflict | El recurso ya existe |
| `InsufficientBalanceException` | 400 Bad Request | La operacion viola una regla de negocio |
| `OrderStatusTransitionException` | 400 Bad Request | Estado invalido para la operacion |
| `EntityNotFoundException` | 404 Not Found | Recurso no encontrado |
| `UnauthorizedActionException` | 403 Forbidden | El usuario no tiene permisos |

El handler tambien debe registrar las excepciones con el nivel de severidad adecuado. Las excepciones de negocio esperadas se registran como warn, no como error. Los errores inesperados (NullPointerException, excepciones de infraestructura) se registran como error porque indican un fallo que no se anticipo.

> No uses excepciones de negocio para control de flujo. Lanzar y capturar excepciones es costoso en terminos de rendimiento. Si una condicion es parte del flujo normal (por ejemplo, verificar si un email existe), usa metodos de consulta como `existsByEmail()` en lugar de lanzar y capturar excepciones.
