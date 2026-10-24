# Form Login

La autenticación por formulario es el mecanismo más utilizado en aplicaciones web tradicionales. Spring Security intercepta las peticiones POST a una URL específica, extrae las credenciales del formulario, las valida contra el `UserDetailsService` configurado y gestiona la sesión del usuario. El flujo incluye redirecciones automáticas a páginas de login, manejo de intentos fallidos y configuración de "recordarme" para sesiones persistentes.

A diferencia de HTTP Basic, que envía credenciales en cada petición, el formulario de login establece una sesión HTTP (JSESSIONID) que se mantiene durante toda la navegación del usuario. Esto es más práctico para aplicaciones web pero requiere consideraciones adicionales sobre gestión de sesiones y protección CSRF.

## Pagina de login personalizada

Spring Security proporciona una página de login por defecto, pero es habitual reemplazarla por una personalizada que se integre con el diseño de la aplicación.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/login").permitAll()
            .requestMatchers("/css/**", "/js/**").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(form -> form
            .loginPage("/login")
            .loginProcessingUrl("/perform-login")
            .usernameParameter("email")
            .passwordParameter("password")
            .defaultSuccessUrl("/dashboard", true)
            .failureUrl("/login?error=true")
            .permitAll()
        );

    return http.build();
}
```

```java
@Controller
public class LoginController {

    @GetMapping("/login")
    public String loginPage() {
        return "login";
    }
}
```

```html
<!-- templates/login.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Iniciar sesion</title>
    <link rel="stylesheet" th:href="@{/css/login.css}" />
</head>
<body>
    <div class="login-container">
        <h2>Iniciar sesion</h2>

        <div th:if="${param.error}" class="alert alert-error">
            Email o password incorrectos
        </div>

        <div th:if="${param.logout}" class="alert alert-success">
            Sesion cerrada correctamente
        </div>

        <form th:action="@{/perform-login}" method="post">
            <div class="form-group">
                <label for="email">Correo electronico</label>
                <input type="email"
                       id="email"
                       name="email"
                       required
                       autofocus
                       placeholder="tu@email.com" />
            </div>

            <div class="form-group">
                <label for="password">Password</label>
                <input type="password"
                       id="password"
                       name="password"
                       required
                       placeholder="Tu password" />
            </div>

            <div class="form-group">
                <label>
                    <input type="checkbox" name="remember-me" /> Recordarme
                </label>
            </div>

            <button type="submit" class="btn-primary">Ingresar</button>
        </form>
    </div>
</body>
</html>
```

## Success y failure handlers

Los handlers personalizados permiten ejecutar lógica adicional después de un login exitoso o fallido, como registrar eventos, redirigir a URLs específicas según el rol o incrementar contadores de intentos.

```java
@Component
public class CustomAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                       HttpServletResponse response,
                                       Authentication authentication) throws IOException {

        Set<String> roles = authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toSet());

        String destino;
        if (roles.contains("ROLE_ADMIN")) {
            destino = "/admin/dashboard";
        } else if (roles.contains("ROLE_VENDEDOR")) {
            destino = "/ventas/dashboard";
        } else {
            destino = "/dashboard";
        }

        response.sendRedirect(destino);
    }
}
```

```java
@Component
public class CustomAuthenticationFailureHandler implements AuthenticationFailureHandler {

    private final IntentoLoginService intentoLoginService;

    public CustomAuthenticationFailureHandler(IntentoLoginService intentoLoginService) {
        this.intentoLoginService = intentoLoginService;
    }

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
                                       HttpServletResponse response,
                                       AuthenticationException exception) throws IOException {

        String email = request.getParameter("email");
        intentoLoginService.registrarIntentoFallido(email);

        if (intentoLoginService.estaBloqueado(email)) {
            response.sendRedirect("/login?blocked=true");
        } else {
            response.sendRedirect("/login?error=true");
        }
    }
}
```

```java
@Service
public class IntentoLoginService {
    private final Map<String, AtomicInteger> intentos = new ConcurrentHashMap<>();

    public void registrarIntentoFallido(String email) {
        intentos.computeIfAbsent(email, k -> new AtomicInteger(0)).incrementAndGet();
    }

    public boolean estaBloqueado(String email) {
        AtomicInteger contador = intentos.get(email);
        return contador != null && contador.get() >= 5;
    }

    public void limpiarIntentos(String email) {
        intentos.remove(email);
    }
}
```

## Remember me

"Remember me" permite a los usuarios mantener su sesión activa entre visitas, almacenando un token persistente en una cookie del navegador.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .rememberMe(remember -> remember
            .key("clave-secreta-unico")
            .tokenValiditySeconds(86400 * 30) // 30 dias
            .rememberMeParameter("remember-me")
            .userDetailsService(userDetailsService)
        );

    return http.build();
}
```

```java
// Remember me con base de datos (persistente)
@Bean
public PersistentTokenRepository persistentTokenRepository(DataSource dataSource) {
    JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
    tokenRepository.setDataSource(dataSource);
    tokenRepository.setCreateTableOnStartup(false); // true para la primera vez
    return tokenRepository;
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .rememberMe(remember -> remember
            .tokenRepository(persistentTokenRepository(dataSource))
            .tokenValiditySeconds(86400 * 30)
        );

    return http.build();
}
```

```sql
-- Tabla para persistent tokens (generada por Spring Security)
CREATE TABLE persistent_logins (
    username  VARCHAR(64) NOT NULL,
    series    VARCHAR(64) PRIMARY KEY,
    token     VARCHAR(64) NOT NULL,
    last_used TIMESTAMP NOT NULL
);
```

> El token "remember me" se almacena como cookie en el navegador. Si un atacante roba esta cookie, puede suplantar al usuario. Para mayor seguridad, se puede combinar con la dirección IP del cliente o con un segundo factor de autenticación.

El formulario de login es el mecanismo de autenticación más flexible de Spring Security. Su configuración permite adaptarse a prácticamente cualquier requisito de UX, desde páginas de login simples hasta flujos complejos con bloqueo de cuentas, handlers personalizados y sesiones persistentes.

