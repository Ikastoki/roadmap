# Security Best Practices

La seguridad de una aplicación no se logra configurando un framework correctamente; requiere una mentalidad de defensa en profundidad donde cada capa de la arquitectura contribuya a reducir la superficie de ataque. Spring Security proporciona las herramientas, pero su efectividad depende de cómo se apliquen. Las mejores prácticas de seguridad cubren desde el almacenamiento de contraseñas hasta la configuración de headers HTTP, pasando por la aplicación del principio de menor privilegio.

Un error común es confiar exclusivamente en las protecciones por defecto de Spring Security. Aunque estas son sólidas, no cubren todos los escenarios. Cada aplicación tiene requisitos de seguridad específicos que deben evaluarse y configurarse de forma explícita.

## HTTPS forzado

Forzar HTTPS garantiza que toda la comunicación entre el cliente y el servidor esté cifrada. Spring Security puede redirigir automáticamente todas las peticiones HTTP a HTTPS.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .requiresChannel(channel -> channel
            .anyRequest().requiresSecure()
        )
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        );

    return http.build();
}
```

```yaml
# application.yml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}
    key-store-type: PKCS12
```

```java
// En produccion detras de un proxy (Nginx, Apache)
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
                .preload(true)
            )
        );

    return http.build();
}
```

> Cuando la aplicación está detrás de un proxy reverso, Spring Security debe configurarse para confiar en los headers `X-Forwarded-Proto` para determinar si la conexión original era HTTPS. Spring Boot habilita esto automáticamente con `server.forward-headers-strategy=NATIVE`.

## Content Security Policy

CSP (Content Security Policy) previene ataques XSS al restringir qué recursos (scripts, estilos, imágenes, fuentes) puede cargar la aplicación.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self' https://cdn.jsdelivr.net; " +
                    "style-src 'self' 'unsafe-inline'; " +
                    "img-src 'self' data: https:; " +
                    "font-src 'self' https://fonts.gstatic.com; " +
                    "connect-src 'self' https://api.miapp.com"
                )
            )
        );

    return http.build();
}
```

```yaml
# Configuracion en application.yml
spring:
  security:
    headers:
      content-security-policy:
        policy-directives: "default-src 'self'; script-src 'self'"
```

## Security Headers

Spring Security incluye múltiples headers de seguridad que protegen contra diversas formas de ataque.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            .contentTypeOptions(Customizer.withDefaults())    // X-Content-Type-Options: nosniff
            .frameOptions(frame -> frame.deny())             // X-Frame-Options: DENY
            .xssProtection(Customizer.withDefaults())         // X-XSS-Protection: 0
            .httpStrictTransportSecurity(hsts -> hsts         // HSTS
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)
            )
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
            )
            .permissionsPolicy(permissions -> permissions
                .policy("camera=(), microphone=(), geolocation=()")
            )
        );

    return http.build();
}
```

| Header                    | Proteccion                                         |
|---------------------------|----------------------------------------------------|
| X-Content-Type-Options    | Previene MIME sniffing                              |
| X-Frame-Options           | Previene clickjacking                               |
| X-XSS-Protection          | Proteccion XSS basica en navegadores antiguos       |
| Strict-Transport-Security | Fuerza HTTPS en el navegador                        |
| Referrer-Policy           | Controla la informacion enviada en Referer          |
| Permissions-Policy        | Restringe acceso a APIs del navegador               |

> `X-XSS-Protection: 0` se recomienda en lugar de `1` porque el filtro XSS del navegador puede introducir vulnerabilidades. CSP es una protección más robusta y moderna.

## Password storage con BCrypt

Nunca almacenar contraseñas en texto plano. BCrypt es el algoritmo recomendado porque incluye salt automático y es intencionalmente lento para resistir ataques de fuerza bruta.

```java
@Configuration
public class PasswordConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Cost factor 12
    }
}
```

```java
// Cifrar contraseñas
@Service
public class UsuarioService {
    private final PasswordEncoder passwordEncoder;

    public void cambiarPassword(String email, String nuevaPassword) {
        Usuario usuario = usuarioRepository.findByEmail(email)
                .orElseThrow();

        // Nunca guardar password en texto plano
        usuario.setPassword(passwordEncoder.encode(nuevaPassword));
        usuarioRepository.save(usuario);
    }

    public boolean verificarPassword(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}
```

> El cost factor de BCrypt determina cuántas iteraciones se ejecutan al cifrar. Un valor de 12 es un buen equilibrio entre seguridad y rendimiento. Valores mayores aumentan la resistencia a fuerza bruta pero ralenticen el login.

## Principle of least privilege

Cada componente de la aplicación debe tener solo los permisos mínimos necesarios para realizar su función. Esto aplica tanto a usuarios como a servicios y configuraciones de base de datos.

```java
// Roles granulares en lugar de roles monoliticos
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.withRolePrefix("ROLE_")
            .role("SUPER_ADMIN").implies("ADMIN")
            .role("ADMIN").implies("GERENTE")
            .role("GERENTE").implies("USUARIO")
            .role("USUARIO").implies("LECTOR")
            .build();
}

// Permisos granulares por recurso
.requestMatchers(HttpMethod.GET, "/api/productos").hasAnyRole("LECTOR", "USUARIO", "ADMIN")
.requestMatchers(HttpMethod.POST, "/api/productos").hasAnyRole("USUARIO", "ADMIN")
.requestMatchers(HttpMethod.DELETE, "/api/productos/**").hasRole("ADMIN")

// Servicios con permisos minimos
@Service
public class ReporteService {

    @PreAuthorize("hasAuthority('REPORTES_LEER')")
    public List<Reporte> generarReportes() {
        // Solo lectura, sin permisos de escritura
    }
}
```

```java
// Base de datos: usuario con permisos minimos
spring:
  datasource:
    username: app_readonly_user
    password: ${DB_PASSWORD}
    # Este usuario solo tiene SELECT, no INSERT/UPDATE/DELETE
```

```yaml
# Separate datasources for read and write
spring:
  datasource:
    read:
      url: jdbc:postgresql://replica:5432/mi_db
      username: readonly_user
    write:
      url: jdbc:postgresql://master:5432/mi_db
      username: admin_user
```

## Validacion de entradas

La validación de entradas previene inyección SQL, XSS y otros ataques basados en datos maliciosos.

```java
public record ProductoDTO(
    @NotBlank @Size(min = 2, max = 100) String nombre,
    @NotNull @DecimalMin("0.01") BigDecimal precio,
    @Size(max = 500) String descripcion
) {}

@RestController
public class ProductoController {

    @PostMapping("/api/productos")
    public ResponseEntity<Producto> crear(
            @RequestBody @Valid ProductoDTO dto) {
        // Los datos estan validados antes de llegar al servicio
    }
}
```

La seguridad es un proceso continuo, no un estado que se alcanza una vez. Las vulnerabilidades se descubren constantemente, los proveedores publican actualizaciones de seguridad y los atacantes desarrollan nuevas técnicas. Mantener las dependencias actualizadas, auditar los permisos periódicamente y monitorizar los logs de seguridad son prácticas esenciales que complementan la configuración técnica.

