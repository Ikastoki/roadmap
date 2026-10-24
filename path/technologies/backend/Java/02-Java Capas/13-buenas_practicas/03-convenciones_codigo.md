# Convenciones de codigo en proyectos por capas

Las convenciones de codigo son el pegamento que mantiene un proyecto comprensible a medida que crece. En una arquitectura por capas, donde la intencion de cada clase debe ser evidente solo por su nombre y ubicacion, seguir una nomenclatura coherente no es un lujo estetico sino una necesidad practica: reduce el tiempo de incorporacion de nuevos desarrolladores, elimina ambiguedades sobre donde colocar nueva funcionalidad y permite que las herramientas de busqueda y navegacion del IDE funcionen de manera predecible.

## Naming de clases segun su rol arquitectonico

Cada capa utiliza sufijos que identifican inequivocamente el rol de la clase dentro de la arquitectura:

- **Controller**: maneja peticiones HTTP y delega en servicios. Ej: `UsuarioController`, `PedidoController`.
- **Service**: contiene logica de negocio u orquestacion. Ej: `UsuarioService`, `FacturacionService`.
- **Repository**: abstrae el acceso a datos. Ej: `UsuarioRepository`, `PedidoRepository`.
- **DTO**: transporta datos entre capas sin exponer entidades. Ej: `UsuarioDTO`, `CrearPedidoRequest`.
- **Mapper**: convierte entre entidades y DTOs. Ej: `UsuarioMapper`, `PedidoMapper`.
- **Exception**: excepcion personalizada del dominio. Ej: `RecursoNoEncontradoException`, `NegocioException`.

```java
// Convencion de nombres clara
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioController { /* ... */ }

public interface UsuarioService { /* ... */ }

public class UsuarioServiceImpl implements UsuarioService { /* ... */ }

public interface UsuarioRepository extends JpaRepository<Usuario, Long> { /* ... */ }

public record UsuarioDTO(Long id, String email, String nombre) { /* ... */ }
```

Los nombres de los metodos deben reflejar la accion que realizan sin ambiguedad. Para servicios y repositorios se recomienda usar verbos de dominio: `buscarPorId`, `crearPedido`, `cancelarReserva`. Para controladores se sigue la convencion REST: `GET /usuarios/{id}` se asigna a `obtener(@PathVariable Long id)`.

> No uses sufijos como `Manager`, `Helper`, `Util` o `Handler` de forma generica. Si una clase no encaja en Controller, Service o Repository, reconsidera si su responsabilidad esta bien definida antes de esconderla tras un nombre ambiguo.

## Estructura de paquetes: por capa o por dominio

Existen dos enfoques principales para organizar los paquetes en un proyecto Java con arquitectura por capas, cada uno con sus ventajas y compensaciones.

**Organizacion por capa**:

```
com.empresa.app/
  controller/
    UsuarioController.java
    PedidoController.java
  service/
    UsuarioService.java
    PedidoService.java
  repository/
    UsuarioRepository.java
    PedidoRepository.java
  model/
    Usuario.java
    Pedido.java
  dto/
    UsuarioDTO.java
    PedidoDTO.java
```

Este enfoque agrupa todas las clases del mismo tipo arquitectonico. Es sencillo de entender y facil de navegar cuando se sabe que tipo de clase se busca. Sin embargo, cuando el dominio crece, cada paquete acumula decenas de clases no relacionadas entre si, y una funcionalidad transversal (ej: "gestion de usuarios") queda dispersa en cinco paquetes diferentes.

**Organizacion por dominio** (recomendada):

```
com.empresa.app/
  usuario/
    UsuarioController.java
    UsuarioService.java
    UsuarioRepository.java
    Usuario.java
    UsuarioDTO.java
    UsuarioMapper.java
  pedido/
    PedidoController.java
    PedidoService.java
    PedidoRepository.java
    Pedido.java
    PedidoDTO.java
    PedidoMapper.java
  infraestructura/
    config/       -- configuracion global
    exception/    -- manejadores globales de excepciones
    security/     -- seguridad transversal
```

En esta variante, cada paquete contiene todas las capas para un agregado de dominio concreto. La cohesion es alta: todo lo relativo a "usuario" esta en el mismo lugar. Es la organizacion preferida en proyectos medianos y grandes porque escala mejor y refleja la estructura del dominio, no la de la tecnologia. Spring Boot la soporta de forma natural gracias al escaneo de componentes.

> La organizacion por dominio es la recomendada por Eric Evans en Domain-Driven Design y es la que utiliza la mayoria de proyectos Spring Boot profesionales. La organizacion por capa solo tiene sentido en proyectos muy pequenos o como paso intermedio durante una migracion.

## Manejo de excepciones centralizado

Las excepciones deben ser especificas del dominio y manejarse de manera centralizada para evitar la duplicacion de logica de error en cada controlador.

**Excepciones personalizadas**:

```java
public abstract class NegocioException extends RuntimeException {
    private final String codigo;
    
    public NegocioException(String codigo, String mensaje) {
        super(mensaje);
        this.codigo = codigo;
    }
    
    public String getCodigo() { return codigo; }
}

public class RecursoNoEncontradoException extends NegocioException {
    public RecursoNoEncontradoException(String recurso, Object id) {
        super("RECURSO_NO_ENCONTRADO", 
              recurso + " con id " + id + " no encontrado");
    }
}

public class ReglaNegocioVioladaException extends NegocioException {
    public ReglaNegocioVioladaException(String descripcion) {
        super("REGLA_VIOLADA", descripcion);
    }
}
```

**Manejador global** con `@ControllerAdvice`:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(RecursoNoEncontradoException.class)
    public ResponseEntity<ErrorDTO> handleNotFound(RecursoNoEncontradoException ex) {
        ErrorDTO error = new ErrorDTO(ex.getCodigo(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(ReglaNegocioVioladaException.class)
    public ResponseEntity<ErrorDTO> handleBusinessRule(ReglaNegocioVioladaException ex) {
        ErrorDTO error = new ErrorDTO(ex.getCodigo(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorDTO> handleValidation(MethodArgumentNotValidException ex) {
        String mensaje = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
            .body(new ErrorDTO("VALIDACION", mensaje));
    }
}
```

Este patron evita que los controladores tengan bloques `try-catch` y concentra la logica de mapeo de excepciones a codigos HTTP en un solo lugar, facilitando la consistencia en las respuestas de error de toda la API.

## Logging estrategico

El logging no debe usarse como sustituto del debugger ni como bitacora de cada operacion interna. Un buen logging en capas sigue estas pautas:

- **Capa de presentacion**: loguea la entrada y salida de peticiones HTTP (metodo, URI, codigo de respuesta, duracion). Spring Boot lo proporciona mediante `spring.jpa.show-sql` o filtros personalizados.
- **Capa de negocio**: loguea decisiones de negocio relevantes: "Usuario X bloqueado por superar intentos", "Pedido Y marcado como fraudulento". No loguees cada llamada a metodo.
- **Capa de persistencia**: loguea consultas lentas (por encima de un umbral configurable) y errores de base de datos.

```java
@Service
public class UsuarioService {
    private static final Logger log = LoggerFactory.getLogger(UsuarioService.class);
    
    public void bloquearUsuario(Long usuarioId) {
        Usuario usuario = repository.findById(usuarioId)
            .orElseThrow(() -> new RecursoNoEncontradoException("Usuario", usuarioId));
        usuario.bloquear();
        repository.save(usuario);
        log.warn("Usuario {} bloqueado por intentos fallidos", usuarioId);
    }
}
```

> Los niveles de log deben usarse con criterio: `ERROR` para condiciones que requieren intervencion, `WARN` para situaciones anomales pero recuperables, `INFO` para eventos de negocio significativos, `DEBUG` para detalle de diagnostico (nunca en produccion por defecto). Evita `log.info()` en cada metodo: satura los logs y oculta la informacion realmente util.
