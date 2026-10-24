# Validation

Bean Validation (JSR 380) es un estándar Java para la validación de objetos mediante anotaciones en sus campos y propiedades. Spring MVC integra Bean Validation a través de Hibernate Validator, proporcionando validación automática de datos de entrada en controllers. Esta integración permite validar objetos antes de que lleguen a la lógica de negocio, mejorando la integridad de los datos y la experiencia del usuario.

`@Valid` y `@Validated` son las anotaciones que activan la validación. `@Valid` es la anotación estándar de Bean Validation que ejecuta todas las restricciones definidas en el objeto. `@Validated` es una anotación de Spring que soporta grupos de validación, permitiendo validaciones condicionales según el contexto. En controllers REST, `@Valid` se usa típicamente en parámetros anotados con `@RequestBody`.

```java
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioController {
    
    @PostMapping
    public ResponseEntity<Usuario> crear(@RequestBody @Valid Usuario usuario) {
        Usuario creado = usuarioService.crear(usuario);
        return ResponseEntity.created(URI.create("/api/usuarios/" + creado.getId()))
            .body(creado);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<Usuario> actualizar(@PathVariable Long id,
                                             @RequestBody @Validated(ActualizarGroup.class) Usuario usuario) {
        Usuario actualizado = usuarioService.actualizar(id, usuario);
        return ResponseEntity.ok(actualizado);
    }
}
```

Las anotaciones de validación más comunes incluyen `@NotNull` para campos no nulos, `@NotEmpty` para colecciones y strings no vacíos, y `@NotBlank` para strings que no pueden estar en blanco. Estas anotaciones garantizan que los campos obligatorios estén presentes y tengan valores válidos.

`@Size` define la longitud mínima y máxima para strings y colecciones. `@Min` y `@Max` definen rangos numéricos para campos numéricos. Estas anotaciones son fundamentales para validar datos que tienen restricciones de tamaño específicas.

```java
public class Usuario {
    
    @NotNull(message = "El ID no puede ser nulo")
    @Min(value = 1, message = "El ID debe ser mayor a 0")
    private Long id;
    
    @NotBlank(message = "El nombre es obligatorio")
    @Size(min = 2, max = 100, message = "El nombre debe tener entre 2 y 100 caracteres")
    private String nombre;
    
    @NotBlank(message = "El email es obligatorio")
    @Email(message = "El email debe ser válido")
    private String email;
    
    @Size(max = 255, message = "La biografía no puede exceder 255 caracteres")
    private String biografia;
    
    @NotNull(message = "La edad es obligatoria")
    @Min(value = 18, message = "La edad mínima es 18 años")
    @Max(value = 120, message = "La edad máxima es 120 años")
    private Integer edad;
}
```

`@Email` valida que el campo contenga una dirección de email válida según el estándar RFC 2822. `@Pattern` permite definir expresiones regulares personalizadas para validación. Estas anotaciones son esenciales para validar formatos de datos específicos que no tienen anotaciones dedicadas.

```java
public class Registro {
    
    @NotBlank
    @Email
    private String email;
    
    @NotBlank
    @Pattern(regexp = "^[A-Z]{2}\\d{8}$", message = "El pasaporte debe tener formato XX12345678")
    private String pasaporte;
    
    @NotBlank
    @Pattern(regexp = "^\\+?\\d{10,15}$", message = "El teléfono debe ser válido")
    private String telefono;
    
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).{8,}$", 
             message = "La contraseña debe tener 8+ caracteres, mayúscula, minúscula y número")
    private String password;
}
```

Los validadores personalizados permiten crear restricciones de validación complejas que no pueden expresarse con las anotaciones estándar. Se definen mediante una anotación personalizada y una clase que implementa `ConstraintValidator`. Esta combinación proporciona validación reutilizable y mantenible para reglas de negocio específicas.

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = FechaFuturaValidator.class)
public @interface FechaFutura {
    String message() default "La fecha debe ser en el futuro";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class FechaFuturaValidator implements ConstraintValidator<FechaFutura, LocalDate> {
    
    @Override
    public boolean isValid(LocalDate fecha, ConstraintValidatorContext context) {
        if (fecha == null) {
            return true; // @NotNull se encarga de validar nulos
        }
        return fecha.isAfter(LocalDate.now());
    }
}
```

Los grupos de validación permiten ejecutar diferentes conjuntos de restricciones según el contexto. Por ejemplo, al crear un usuario se requiere contraseña, pero al actualizar no. Los grupos se definen como interfaces vacías y se seleccionan usando `@Validated`.

```java
public interface CrearGroup {}
public interface ActualizarGroup {}

public class Usuario {
    
    @NotBlank(groups = CrearGroup.class, message = "La contraseña es obligatoria al crear")
    @Size(min = 8, groups = CrearGroup.class)
    private String password;
    
    @NotBlank(message = "El nombre es obligatorio")
    private String nombre;
}

@RestController
public class UsuarioController {
    
    @PostMapping
    public ResponseEntity<Usuario> crear(@RequestBody @Validated(CrearGroup.class) Usuario usuario) {
        return ResponseEntity.ok(usuarioService.crear(usuario));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<Usuario> actualizar(@PathVariable Long id,
                                             @RequestBody @Validated(ActualizarGroup.class) Usuario usuario) {
        return ResponseEntity.ok(usuarioService.actualizar(id, usuario));
    }
}
```

`BindingResult` contiene los errores de validación y se puede usar en lugar de `@Valid` para manejar errores manualmente. Cuando `BindingResult` está presente como parámetro del método, Spring no lanza `MethodArgumentNotValidException` y permite al desarrollador decidir cómo manejar los errores. Esto es útil para formularios donde se quiere mostrar errores específicos al usuario.

```java
@Controller
@RequestMapping("/usuarios")
public class UsuarioFormController {
    
    @PostMapping("/crear")
    public String crear(@ModelAttribute @Valid Usuario usuario, 
                       BindingResult result, Model model) {
        if (result.hasErrors()) {
            model.addAttribute("errores", result.getAllErrors());
            return "usuario-form";
        }
        usuarioService.crear(usuario);
        return "redirect:/usuarios";
    }
}
```

El manejo de errores de validación en APIs REST se realiza típicamente mediante `@ExceptionHandler` que captura `MethodArgumentNotValidException`. Esta excepción contiene todos los errores de validación y permite construir una respuesta estructurada con detalles específicos de cada error.

```java
@RestControllerAdvice
public class ValidacionExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> manejarValidacion(
            MethodArgumentNotValidException ex) {
        
        Map<String, Object> errores = new HashMap<>();
        List<Map<String, String>> detalles = new ArrayList<>();
        
        ex.getBindingResult().getFieldErrors().forEach(error -> {
            Map<String, String> detalle = new HashMap<>();
            detalle.put("campo", error.getField());
            detalle.put("mensaje", error.getDefaultMessage());
            detalles.add(detalle);
        });
        
        errores.put("timestamp", LocalDateTime.now());
        errores.put("status", HttpStatus.BAD_REQUEST.value());
        errores.put("errores", detalles);
        
        return ResponseEntity.badRequest().body(errores);
    }
}
```

> La validación debe ser la primera línea de defensa contra datos inválidos, pero no debe ser la única. La lógica de negocio también debe validar datos, ya que las restricciones de Bean Validation son estáticas y pueden no cubrir todas las reglas de validación dinámicas.

