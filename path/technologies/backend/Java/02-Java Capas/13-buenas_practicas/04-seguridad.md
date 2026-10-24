# Seguridad en la arquitectura por capas

La seguridad no es una capa adicional que se anade al final del desarrollo, sino una propiedad transversal que debe integrarse en cada nivel de la arquitectura. Un sistema en capas bien disenado proporciona puntos naturales donde aplicar controles de seguridad: la capa de presentacion valida y filtra la entrada, la capa de negocio aplica reglas de autorizacion, y la capa de persistencia protege el acceso a los datos. Ignorar cualquiera de estos puntos compromete la seguridad del conjunto.

## Validacion de entrada en la capa de presentacion

Toda entrada que llega desde el exterior --parametros de peticion, cuerpos JSON, cabeceras HTTP, variables de ruta-- debe considerarse potencialmente hostil hasta que se demuestre lo contrario. La validacion debe ocurrir en la capa de presentacion, antes de que los datos lleguen a la logica de negocio.

Spring Boot proporciona Bean Validation (JSR 380) mediante anotaciones en los DTOs de entrada:

```java
public record CrearUsuarioRequest(
    @NotBlank(message = "El email es obligatorio")
    @Email(message = "Formato de email invalido")
    String email,
    
    @NotBlank(message = "El nombre es obligatorio")
    @Size(min = 2, max = 100, message = "El nombre debe tener entre 2 y 100 caracteres")
    String nombre,
    
    @NotNull
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[0-9]).{8,}$", 
             message = "La contrasena debe tener al menos 8 caracteres, una mayuscula y un numero")
    String contrasena
) {}
```

La validacion se activa anadiendo `@Valid` en el controlador:

```java
@PostMapping("/usuarios")
public ResponseEntity<UsuarioDTO> crear(@RequestBody @Valid CrearUsuarioRequest request) {
    // El request ya esta validado; si no, lanza MethodArgumentNotValidException
    UsuarioDTO usuario = usuarioService.crear(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(usuario);
}
```

> La validacion en el controlador es la primera barrera, pero no la unica. La capa de negocio debe realizar sus propias validaciones semanticas (ej: "el email ya esta registrado") que no pueden expresarse con anotaciones. Nunca confies unicamente en la validacion de presentacion.

Ademas de validar, la capa de presentacion debe sanitizar la entrada para prevenir ataques de inyeccion. Aunque un ORM como JPA protege contra inyeccion SQL cuando se usan consultas parametrizadas, otras vulnerabilidades como XSS requieren escape activo de la salida si la aplicacion genera HTML.

## No exponer entidades: el rol de los DTOs

Uno de los errores de seguridad mas comunes en aplicaciones por capas es exponer directamente las entidades JPA en las respuestas HTTP. Esto filtra informacion interna que el cliente no deberia conocer: IDs de auditoria, campos internos, relaciones con otras entidades que se serializan en cascada, o datos sensibles como hashes de contrasenas.

```java
// Mal: exponer la entidad directamente
@GetMapping("/usuarios/{id}")
public Usuario obtener(@PathVariable Long id) {
    return usuarioService.buscarPorId(id); // Expone contrasenaHash, fechaCreacion, etc.
}

// Bien: usar un DTO
@GetMapping("/usuarios/{id}")
public UsuarioDTO obtener(@PathVariable Long id) {
    return usuarioService.buscarPorId(id); // Devuelve solo los campos del DTO
}
```

Los DTOs de respuesta deben contener exclusivamente los datos que el cliente necesita, ni mas ni menos:

```java
public record UsuarioDTO(
    Long id,
    String email,
    String nombre,
    boolean activo
) {}

// La entidad puede tener muchos mas campos que no se exponen
@Entity
public class Usuario {
    @Id private Long id;
    private String email;
    private String nombre;
    private boolean activo;
    private String contrasenaHash;   // No debe salir en la respuesta
    private LocalDateTime fechaCreacion; // Tampoco
    @ManyToMany private Set<Rol> roles; // Las colecciones perezosas lanzan LazyInitializationException
}
```

El uso consistente de DTOs protege contra la sobreexposicion accidental de datos y evita problemas de serializacion con proxies de Hibernate. Ademas, al desacoplar el contrato API del modelo de persistencia, permite evolucionar ambos de forma independiente.

> Si usas `@JsonIgnore` o `@JsonProperty(access = Access.WRITE_ONLY)` en la entidad como sustituto de los DTOs, estas creando un acoplamiento peligroso entre tu capa de persistencia y tu API. Un cambio en la entidad puede filtrar datos sin que el compilador advierta del problema.

## Autenticacion y autorizacion

La autenticacion (verificar quien es el usuario) y la autorizacion (verificar que puede hacer) deben implementarse en la capa adecuada:

- **Autenticacion**: ocurre tipicamente en un filtro de la capa de presentacion, antes de que la peticion llegue al controlador. Spring Security utiliza una cadena de filtros (`SecurityFilterChain`) que intercepta la peticion, extrae las credenciales (JWT, sesion, API key) y establece el contexto de seguridad.
- **Autorizacion**: se aplica tanto a nivel de ruta (en la configuracion de seguridad) como a nivel de metodo (en la capa de negocio).

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session.sessionCreationPolicy(STATELESS))
            .oauth2ResourceServer(OAuth2ResourceServerConfigurer::jwt);
        return http.build();
    }
}

// Autorizacion a nivel de metodo en la capa de negocio
@Service
public class PedidoService {
    
    @PreAuthorize("hasRole('ADMIN') or #usuarioId == authentication.principal.id")
    public PedidoDTO cancelarPedido(Long usuarioId, Long pedidoId) {
        // Solo el propietario del pedido o un admin puede cancelarlo
    }
}
```

> La autorizacion en la capa de negocio es la segunda linea de defensa. Un controlador que confia unicamente en la configuracion de rutas puede ser vulnerable si alguien llama al servicio desde otro punto de entrada (ej: un evento de cola, una consola de administracion, un test). La anotacion `@PreAuthorize` garantiza que la regla se evalua independientemente de como se invoque el metodo.

## HTTPS y comunicaciones seguras

La integridad y confidencialidad de los datos en transito no es negociable en entornos productivos. Toda comunicacion entre el cliente y el servidor debe cursarse sobre TLS/HTTPS. En Spring Boot, la configuracion tipica fuerza HTTPS y redirige automaticamente el trafico HTTP:

```properties
# application.properties
server.port=8443
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=changeit
server.ssl.key-store-type=PKCS12
server.ssl.key-alias=tomcat
```

Para entornos de desarrollo o contenedores que terminan SSL en un proxy inverso (nginx, Traefik, AWS ALB), se configura la confianza en las cabeceras `X-Forwarded-*`:

```java
@Bean
public TomcatServletWebServerFactory tomcatFactory() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    RemoteIpValve valve = new RemoteIpValve();
    valve.setProtocolHeader("X-Forwarded-Proto");
    factory.addEngineValves(valve);
    return factory;
}
```

El proxy inverso se encarga de la terminacion SSL, mientras que la aplicacion confia en que la cabecera `X-Forwarded-Proto: https` indica una conexion segura. Spring Boot debe configurarse para respetar esta cabecera y generar URLs con `https` en los enlaces de autorizacion OAuth, HATEOAS, etc.

> HTTPS protege contra ataques de intermediario (MITM), pero no contra vulnerabilidades de la aplicacion. Un endpoint con HTTPS que expone las entidades directamente o que no valida la entrada sigue siendo inseguro. La seguridad en capas significa aplicar controles en cada nivel, no confiar en que el cifrado del transporte resuelve todos los problemas.
