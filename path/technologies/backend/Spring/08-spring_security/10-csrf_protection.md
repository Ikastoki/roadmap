# CSRF Protection

CSRF (Cross-Site Request Forgery) es un ataque donde un sitio malicioso induce al navegador de un usuario autenticado a realizar peticiones no deseadas a la aplicación de destino. El ataque aprovecha que el navegador envía automáticamente las cookies de sesión (JSESSIONID) en cada petición al mismo dominio, por lo que el servidor no puede distinguir entre una petición legítima y una forzada por un tercero. Spring Security incluye protección CSRF por defecto que valida un token en cada petición de modificación.

El escenario típico de CSRF es el siguiente: un usuario inicia sesión en `banco.com` y mantiene la sesión activa. Posteriormente, visita un sitio malicioso que contiene un formulario oculto con una petición POST a `banco.com/transferir`. El navegador envía la petición junto con la cookie de sesión válida, y el servidor la procesa como si fuera legítima. Spring Security previene esto insertando un token CSRF único en cada formulario que el servidor verifica antes de procesar la petición.

## CSRF habilitado por defecto

Spring Security habilita la protección CSRF de forma predeterminada para todas las peticiones que utilicen métodos de modificación (POST, PUT, DELETE, PATCH). Las peticiones GET no requieren token CSRF porque son idempotentes y no modifican estado.

```java
// Configuracion por defecto (implicita)
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());
    // CSRF esta habilitado por defecto

    return http.build();
}
```

```java
// Configuracion explicita con personalizaciones
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
        );

    return http.build();
}
```

En Thymeleaf, el token CSRF se incluye automáticamente en cada formulario:

```html
<form th:action="@{/transferir}" method="post">
    <!-- Thymeleaf incluye automaticamente el campo _csrf -->
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />
    <input type="text" name="monto" />
    <button type="submit">Transferir</button>
</html>
```

> El token CSRF se almacena en la sesión HTTP del usuario. Cada sesión tiene un token único que se regenera periódicamente. Si la sesión expira, el token CSRF también expira y la petición será rechazada.

## Deshabilitar CSRF para REST APIs

Las APIs REST que utilizan autenticación stateless (JWT, OAuth2) no necesitan protección CSRF porque no dependen de cookies de sesión. El envío del token en el header `Authorization` no puede ser provocado por un formulario cross-origin.

```java
// API REST stateless: CSRF innecesario
@Bean
public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        )
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}

// Web application: CSRF habilitado
@Bean
public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/**")
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/login").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());
    // CSRF habilitado por defecto

    return http.build();
}
```

> No se debe deshabilitar CSRF en aplicaciones web que utilicen sesiones HTTP. Solo es seguro deshabilitarlo cuando la autenticación es completamente stateless (JWT en header).

## CSRF Token Repository

Spring Security almacena los tokens CSRF en la sesión HTTP por defecto. Sin embargo, se pueden utilizar repositorios alternativos para diferentes escenarios.

```java
// Cookie-based: util para aplicaciones SPA
@Bean
public CsrfTokenRepository csrfTokenRepository() {
    return CookieCsrfTokenRepository.withHttpOnlyFalse();
}

// Session-based: comportamiento por defecto
@Bean
public CsrfTokenRepository csrfTokenRepository() {
    return HttpSessionCsrfTokenRepository.withHttpFalse();
}
```

```java
// Configuracion con CookieCsrfTokenRepository
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        );

    return http.build();
}
```

Con `CookieCsrfTokenRepository`, el token CSRF se almacena en una cookie en lugar de en la sesión. Esto es útil para aplicaciones SPA (Single Page Application) donde el frontend necesita leer el token CSRF para incluirlo en las peticiones AJAX.

```javascript
// JavaScript: obtener el token CSRF de la cookie
function getCsrfToken() {
    const name = 'XSRF-TOKEN';
    const cookies = document.cookie.split(';');
    for (let cookie of cookies) {
        cookie = cookie.trim();
        if (cookie.startsWith(name + '=')) {
            return decodeURIComponent(cookie.substring(name.length + 1));
        }
    }
    return null;
}

// Enviar el token en cada peticion AJAX
fetch('/api/productos', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-XSRF-TOKEN': getCsrfToken()
    },
    body: JSON.stringify({ nombre: 'Nuevo Producto' })
});
```

La protección CSRF es un mecanismo esencial para aplicaciones web que utilizan sesiones HTTP. Spring la habilita por defecto y la gestiona de forma transparente, pero es fundamental comprender cuándo es necesaria y cuándo se puede deshabilitar de forma segura.

