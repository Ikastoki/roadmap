# Security Best Practices

La seguridad en aplicaciones Spring Boot debe abordarse desde las primeras fases del desarrollo, no como una capa que se agrega al final. Un enfoque de seguridad por defecto (*secure by default*) garantiza que cada componente este protegido desde su creacion y que los controles de seguridad se apliquen de forma consistente a traves de toda la aplicacion.

## Principio de menor privilegio

Cada componente, servicio y usuario debe tener solo los permisos necesarios para realizar su funcion especifica. En el contexto de Spring Security, esto significa configurar reglas de autorizacion lo mas restrictivas posible y ampliarlas solo cuando sea necesario.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults()))
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/webhook/**"))
            .headers(headers -> headers
                .contentSecurityPolicy(csp ->
                    csp.policyDirectives("default-src 'self'")))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));

        return http.build();
    }
}
```

```java
@PreAuthorize("hasRole('ADMIN') or @ownershipCheck.isOwner(#id, authentication)")
@GetMapping("/api/usuarios/{id}")
public ResponseEntity<UsuarioDTO> obtener(@PathVariable Long id) {
    return ResponseEntity.ok(usuarioService.obtenerPorId(id));
}
```

## Validacion de entrada

Toda la entrada que proviene del exterior (HTTP,消息, archivos) debe validarse antes de procesarse. Spring proporciona `@Valid` con Bean Validation para validar automaticamente los DTOs de entrada.

```java
public class CrearUsuarioDTO {

    @NotBlank(message = "El nombre es obligatorio")
    @Size(min = 2, max = 100, message = "El nombre debe tener entre 2 y 100 caracteres")
    private String nombre;

    @NotBlank(message = "El email es obligatorio")
    @Email(message = "El formato del email no es valido")
    private String email;

    @NotBlank(message = "La contrasena es obligatoria")
    @Size(min = 8, message = "La contrasena debe tener al menos 8 caracteres")
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$",
        message = "La contrasena debe contener mayuscula, minuscula, numero y caracter especial"
    )
    private String contrasena;
}
```

```java
@Component
public class InputSanitizer {

    private static final Pattern HTML_PATTERN = Pattern.compile("<[^>]*>");

    public String sanitize(String input) {
        if (input == null) return null;
        return HTML_PATTERN.matcher(input).replaceAll("")
            .trim();
    }

    public <T> T sanitize(T dto) {
        // Sanitizar campos String del DTO usando reflection
        Arrays.stream(dto.getClass().getDeclaredFields())
            .filter(f -> f.getType() == String.class)
            .forEach(f -> {
                f.setAccessible(true);
                try {
                    String valor = (String) f.get(dto);
                    if (valor != null) {
                        f.set(dto, sanitize(valor));
                    }
                } catch (IllegalAccessException e) {
                    // ignorar
                }
            });
        return dto;
    }
}
```

> La validacion de entrada en el cliente (frontend) es una conveniencia para el usuario, no una medida de seguridad. Siempre se debe re-validar en el servidor porque el cliente puede ser manipulado.

## Proteccion CSRF

Cross-Site Request Forgery (CSRF) es un ataque que fuerza al usuario autenticado a realizar acciones no deseadas. Spring Security habilita la proteccion CSRF por defecto para aplicaciones con sesiones. Para APIs REST con autenticacion stateless (JWT), CSRF se desactiva porque cada peticion incluye un token de autenticacion.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // Para APIs REST stateless - desactivar CSRF
        .csrf(csrf -> csrf.disable())

        // Para aplicaciones web con sesiones - habilitar CSRF
        // .csrf(csrf -> csrf
        //     .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))

        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        );
    return http.build();
}
```

## HTTPS y headers de seguridad

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .requiresChannel(channel -> channel
            .anyRequest().requiresSecure()
        )
        .headers(headers -> headers
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)
            )
            .contentSecurityPolicy(csp ->
                csp.policyDirectives("default-src 'self'; script-src 'self'"))
            .frameOptions(frame -> frame.deny())
            .contentTypeOptions(Customizer.withDefaults())
        );
    return http.build();
}
```

## Cifrado de contrasenas con BCrypt

Nunca almacenar contrasenas en texto plano. BCrypt es el algoritmo recomendado porque incluye salt automatico y es resistente a ataques de fuerza bruta.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // Cost factor 12
}

@Service
public class UsuarioService {

    private final UsuarioRepository repository;
    private final PasswordEncoder passwordEncoder;

    public UsuarioService(UsuarioRepository repository,
                         PasswordEncoder passwordEncoder) {
        this.repository = repository;
        this.passwordEncoder = passwordEncoder;
    }

    public UsuarioDTO crear(CrearUsuarioDTO dto) {
        Usuario usuario = new Usuario();
        usuario.setNombre(dto.getNombre());
        usuario.setEmail(dto.getEmail());
        usuario.setContrasenaHash(passwordEncoder.encode(dto.getContrasena()));

        return toDTO(repository.save(usuario));
    }

    public boolean verificarContrasena(String plano, String hash) {
        return passwordEncoder.matches(plano, hash);
    }
}
```

## Actualizacion regular de dependencias

Las vulnerabilidades de seguridad se descubren diariamente en librerias de terceros. Mantener las dependencias actualizadas es una medida de seguridad critica.

```bash
# Maven
mvn versions:display-dependency-updates

# Gradle
gradle dependencyUpdates

# Escaneo de vulnerabilidades
mvn org.owasp:dependency-check-maven:check
```

```xml
<!-- OWASP Dependency Check Plugin -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>8.4.0</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFiles>
            <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
        </suppressionFiles>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

La seguridad no es un feature que se completa, es un proceso continuo que requiere revisiones de codigo, escaneos de vulnerabilidades periodicos y actualizacion constante de dependencias.

