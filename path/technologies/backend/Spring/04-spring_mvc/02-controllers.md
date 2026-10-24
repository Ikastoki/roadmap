# Controllers

Los controllers son componentes Spring que manejan peticiones HTTP y retornan respuestas. En Spring MVC, existen dos tipos principales: `@Controller` para aplicaciones MVC tradicionales con vistas, y `@RestController` para APIs REST que retornan datos directamente. La diferencia fundamental es que `@RestController` incluye implícitamente `@ResponseBody` en cada método, eliminando la necesidad de anotaciones adicionales.

`@Controller` se usa en aplicaciones que retornan nombres de vistas que serán resueltas por un `ViewResolver`. Cada método puede tener `@ResponseBody` para retornar datos directamente, pero esto no es lo habitual. `@RestController` combina `@Controller` y `@ResponseBody`, haciendo que todos los métodos retornen datos serializados en el body de la respuesta.

```java
// Controller MVC tradicional
@Controller
@RequestMapping("/usuarios")
public class UsuarioMvcController {
    
    @GetMapping("/{id}")
    public String obtenerUsuario(@PathVariable Long id, Model model) {
        model.addAttribute("usuario", usuarioService.obtenerPorId(id));
        return "usuario"; // Nombre de vista a resolver
    }
}

// Controller REST
@RestController
@RequestMapping("/api/usuarios")
public class UsuarioRestController {
    
    @GetMapping("/{id}")
    public Usuario obtenerUsuario(@PathVariable Long id) {
        return usuarioService.obtenerPorId(id); // Serializado a JSON automáticamente
    }
}
```

`@RequestMapping` es la anotación base para mapear peticiones HTTP a métodos del controller. Puede aplicarse a nivel de clase para definir un prefijo de URL común, y a nivel de método para definir la ruta específica. Soporta múltiples atributos como `method`, `params`, `headers` y `consumes`/`produces` para un mapeo más preciso.

Las anotaciones de mapeo de métodos HTTP simplifican `@RequestMapping` para casos comunes. `@GetMapping` mapea peticiones GET, `@PostMapping` mapea peticiones POST, `@PutMapping` mapea peticiones PUT, `@DeleteMapping` mapea peticiones DELETE y `@PatchMapping` mapea peticiones PATCH. Estas anotaciones son más concisas y legibles que usar `@RequestMapping` con el atributo `method`.

```java
@RestController
@RequestMapping("/api/productos")
public class ProductoController {
    
    @GetMapping
    public List<Producto> listar() {
        return productoService.listar();
    }
    
    @GetMapping("/{id}")
    public Producto obtener(@PathVariable Long id) {
        return productoService.obtenerPorId(id);
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Producto crear(@RequestBody Producto producto) {
        return productoService.crear(producto);
    }
    
    @PutMapping("/{id}")
    public Producto actualizar(@PathVariable Long id, @RequestBody Producto producto) {
        return productoService.actualizar(id, producto);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void eliminar(@PathVariable Long id) {
        productoService.eliminar(id);
    }
    
    @PatchMapping("/{id}")
    public Producto actualizarParcial(@PathVariable Long id, 
                                     @RequestBody Map<String, Object> campos) {
        return productoService.actualizarParcial(id, campos);
    }
}
```

`@ResponseStatus` establece el código de estado HTTP de la respuesta. Se puede usar en métodos del controller para especificar el estado exitoso, o en excepciones para indicar el estado de error. Esta anotación simplifica la configuración del estado HTTP sin necesidad de usar `ResponseEntity`.

`@ResponseBody` indica que el objeto retornado por el método debe serializarse directamente en el body de la respuesta. En `@RestController` es implícito, pero en `@Controller` se debe especificar explícitamente para métodos que retornan datos.

`@RequestBody` deserializa el body de la petición HTTP en un objeto Java. Spring utiliza HttpMessageConverters (típicamente Jackson para JSON) para realizar la conversión. Esta anotación es fundamental para APIs REST que reciben datos en el body de peticiones POST o PUT.

`@PathVariable` extrae valores de la URL. Los parámetros de ruta se definen en la anotación de mapeo con llaves `{parametro}` y se inyectan en el método del controller. Soporta expresiones regulares para restringir los valores válidos y conversión de tipos automática.

```java
@GetMapping("/usuarios/{id:[0-9]+}")
public Usuario obtener(@PathVariable Long id) {
    return usuarioService.obtenerPorId(id);
}

@GetMapping("/categorias/{nombre}")
public Categoria obtenerPorNombre(@PathVariable String nombre) {
    return categoriaService.obtenerPorNombre(nombre);
}
```

`@RequestParam` extrae parámetros de la query string de la URL. Se usa para parámetros de búsqueda, paginación y filtros. Soporta valores por defecto, valores requeridos y conversión de tipos. Cuando se usa en parámetros de tipo `List` o `Array`, permite recibir múltiples valores para el mismo parámetro.

```java
@GetMapping("/buscar")
public Page<Usuario> buscar(
        @RequestParam String nombre,
        @RequestParam(required = false, defaultValue = "0") int pagina,
        @RequestParam(defaultValue = "10") int tamano) {
    return usuarioService.buscar(nombre, PageRequest.of(pagina, tamano));
}

@GetMapping("/filtrar")
public List<Usuario> filtrar(
        @RequestParam List<String> roles,
        @RequestParam(required = false) String estado) {
    return usuarioService.filtrar(roles, estado);
}
```

`@MatrixVariable` permite recibir variables de matriz de la URL. Las variables de matriz se definen después del punto y coma en la URL y permiten pasar múltiples valores para un mismo parámetro. Esta funcionalidad es menos común pero útil para ciertos escenarios de filtrado.

`@RequestHeader` extrae valores de los headers HTTP de la petición. Es útil para obtener información como el tipo de cliente, tokens de autenticación o headers personalizados de la aplicación.

```java
@GetMapping("/info")
public Map<String, String> info(
        @RequestHeader("User-Agent") String userAgent,
        @RequestHeader(value = "X-Request-Id", required = false) String requestId) {
    Map<String, String> info = new HashMap<>();
    info.put("userAgent", userAgent);
    info.put("requestId", requestId);
    return info;
}
```

`@CookieValue` extrae valores de las cookies de la petición. Permite acceder a cookies específicas sin necesidad de analizar manualmente el header `Cookie`. Soporta conversión de tipos y valores por defecto.

`@ModelAttribute` bindea parámetros de la petición (tanto de query string como del body en formularios) a objetos de modelo. Es útil para formularios HTML donde múltiples campos se agrupan en un objeto. Cuando se usa en el retorno de un método, el objeto se agrega automáticamente al modelo.

```java
@Controller
@RequestMapping("/usuarios")
public class UsuarioFormController {
    
    @PostMapping("/crear")
    public String crearUsuario(@ModelAttribute @Valid Usuario usuario, 
                              BindingResult result) {
        if (result.hasErrors()) {
            return "usuario-form";
        }
        usuarioService.crear(usuario);
        return "redirect:/usuarios/" + usuario.getId();
    }
}
```

`@SessionAttributes` en la anotación de clase especifica qué atributos del modelo deben guardarse en la sesión HTTP. Esto es útil para flujos de múltiples pasos donde se necesita mantener estado entre peticiones. Los atributos se almacenan en la sesión y se restauran automáticamente en peticiones posteriores.

`@SessionAttribute` (sin 's') extrae un atributo de la sesión HTTP. A diferencia de `@SessionAttributes` que es declarativo y está ligado al ciclo de vida del modelo, `@SessionAttribute` accede directamente a la sesión y es útil para obtener valores que no están en el modelo.

```java
@Controller
@SessionAttributes("carrito")
public class CarritoController {
    
    @ModelAttribute("carrito")
    public Carrito carrito() {
        return new Carrito();
    }
    
    @PostMapping("/agregar")
    public String agregarItem(@ModelAttribute Carrito carrito,
                             @SessionAttribute("usuarioId") Long usuarioId) {
        // carrito se guarda automáticamente en la sesión por @SessionAttributes
        return "redirect:/carrito";
    }
}
```

> La elección entre `@Controller` y `@RestController` depende del tipo de aplicación. Para APIs REST puras, `@RestController` es la opción correcta. Para aplicaciones con servido de páginas HTML mediante plantillas, `@Controller` con métodos que retornan nombres de vistas es más apropiado.

