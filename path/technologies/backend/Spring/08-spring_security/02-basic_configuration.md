# Basic Configuration

La configuración de Spring Security se define mediante un bean `SecurityFilterChain` que establece las reglas de acceso, los mecanismos de autenticación y las protecciones aplicables a cada petición. En Spring Security 6, la API se ha simplificado eliminando `WebSecurityConfigurerAdapter` en favor de una configuración funcional basada en lambdas que mejora la legibilidad y la flexibilidad.

Una configuración típica de seguridad incluye definir qué endpoints son públicos, cuáles requieren autenticación, qué mecanismo de login se utiliza (formulario, HTTP Basic, OAuth2) y cómo se gestionan las sesiones y los tokens CSRF. Cada decisión se expresa de forma declarativa en la cadena de filtros.

## @EnableWebSecurity

`@EnableWebSecurity` activa la integración de Spring Security con Spring MVC. Sin esta anotación, la cadena de filtros de seguridad no se registra y la aplicación queda completamente desprotegida. La anotación también habilita la configuración por defecto de Spring Security, que puede ser sobreescrita mediante la definición de un bean `SecurityFilterChain`.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**", "/login", "/css/**", "/js/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .loginProcessingUrl("/perform-login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error=true")
                .permitAll()
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**")
            )
            .cors(cors -> cors.configurationSource(corsConfigurationSource()));

        return http.build();
    }
}
```

> Si no se define ningún bean `SecurityFilterChain`, Spring Boot aplica una configuración por defecto que protege todos los endpoints con autenticación básica y un usuario `user` con contraseña generada aleatoriamente.

## authorizeHttpRequests

`authorizeHttpRequests` es el método principal para definir reglas de autorización a nivel de URL. Cada `requestMatcher` especifica un patrón de URL y la condición de acceso requerida.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            // Endpoints publicos
            .requestMatchers("/").permitAll()
            .requestMatchers("/home").permitAll()
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers(HttpMethod.GET, "/api/productos").permitAll()

            // Endpoints por rol
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .requestMatchers("/api/usuarios/**").hasAnyRole("ADMIN", "USUARIO_ADMIN")
            .requestMatchers("/api/reportes/**").hasAnyAuthority("REPORTES_VER", "ADMIN")

            // Cualquier otra peticion requiere autenticacion
            .anyRequest().authenticated()
        );

    return http.build();
}
```

```java
// patron de URL con variables
.requestMatchers("/api/usuarios/{id}/**")
    .access(new AuthorizationManager.RequestMatcherDelegatingAuthorizationManager() {
        @Override
        public AuthorizationDecision check(Supplier<Authentication> authentication,
                                           RequestAuthorizationContext context) {
            String userId = context.getVariables().get("id");
            Authentication auth = authentication.get();
            boolean tienePermiso = auth.getName().equals(userId) ||
                                   auth.getAuthorities().stream()
                                       .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
            return new AuthorizationDecision(tienePermiso);
        }
    })
```

## formLogin

`formLogin` configura la autenticación basada en formulario, que es el mecanismo más utilizado en aplicaciones web tradicionales. El usuario提交 credenciales mediante un formulario HTML que Spring Security intercepta para validar.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .formLogin(form -> form
            .loginPage("/login")                    // Pagina de login personalizada
            .loginProcessingUrl("/perform-login")   // URL que procesa el formulario
            .usernameParameter("correo")            // Nombre del campo usuario
            .passwordParameter("clave")             // Nombre del campo password
            .defaultSuccessUrl("/dashboard", true)  // Redireccion exitosa
            .failureUrl("/login?error=true")        // Redireccion fallida
            .permitAll()                            // Permite acceso sin autenticar
        );

    return http.build();
}
```

```html
<!-- Pagina de login personalizada -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Login</title></head>
<body>
    <div th:if="${param.error}">
        <p style="color: red;">Credenciales incorrectas</p>
    </div>
    <form th:action="@{/perform-login}" method="post">
        <div>
            <label for="correo">Correo:</label>
            <input type="email" id="correo" name="correo" required />
        </div>
        <div>
            <label for="clave">Password:</label>
            <input type="password" id="clave" name="clave" required />
        </div>
        <button type="submit">Ingresar</button>
    </form>
</body>
</html>
```

## httpBasic

`httpBasic` habilita la autenticación HTTP Basic, donde las credenciales se envían en cada petición mediante el header `Authorization: Basic base64(usuario:contraseña)`. Es útil para APIs internas o servicios machine-to-machine donde el flujo de formulario no aplica.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .httpBasic(basic -> basic
            .realmName("Mi API")
            .authenticationEntryPoint((request, response, authException) -> {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.setContentType("application/json");
                response.getWriter().write("{\"error\": \"No autorizado\"}");
            })
        );

    return http.build();
}
```

> HTTP Basic envía credenciales en cada petición codificadas en Base64 (no cifradas). Sin HTTPS, las credenciales quedan expuestas en texto plano. Siempre se debe usar HTTPS en producción con HTTP Basic.

## CSRF

La protección CSRF (Cross-Site Request Forgery) está habilitada por defecto. Spring Security incluye un token CSRF en cada formulario y lo valida en el servidor para cada petición de modificación (POST, PUT, DELETE).

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            .ignoringRequestMatchers("/api/**")          // Deshabilitar para APIs REST
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        );

    return http.build();
}
```

## CORS

La configuración CORS controla qué dominios externos pueden hacer peticiones a la aplicación. Sin esta configuración, el navegador bloqueará las peticiones cross-origin.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()));

    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000", "https://miapp.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

## Configuracion completa de ejemplo

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final UserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public SecurityConfig(UserDetailsService userDetailsService,
                         PasswordEncoder passwordEncoder) {
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home", "/login", "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/usuarios/**").hasAnyRole("ADMIN", "USUARIO")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .permitAll()
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**")
            )
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout=true")
                .deleteCookies("JSESSIONID")
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Una configuración completa de Spring Security debe equilibrar seguridad y usabilidad. No todas las protecciones son necesarias en todos los contextos: una API interna no necesita formulario de login, y una aplicación web no debería deshabilitar CSRF. La clave está en comprender cada opción y aplicarlas según el caso de uso específico.

