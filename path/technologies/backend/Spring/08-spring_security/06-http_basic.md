# HTTP Basic

HTTP Basic es un mecanismo de autenticación simple donde el cliente envía credenciales en cada petición mediante el header `Authorization: Basic <base64(usuario:contraseña)>`. No establece sesión, no requiere un formulario de login y funciona de forma stateless, lo que lo convierte en una opción natural para APIs internas, servicios microservicios y herramientas de línea de comandos. Sin embargo, las credenciales se transmiten en cada petición codificadas en Base64 (no cifradas), por lo que HTTPS es obligatorio en producción.

El flujo de HTTP Basic comienza cuando el servidor responde con un header `WWW-Authenticate: Basic realm="..."` ante una petición no autenticada. El navegador muestra un diálogo nativo para capturar credenciales, o el cliente REST las incluye directamente en el header.

## Comportamiento por defecto

Al habilitar HTTP Basic, Spring Security valida las credenciales del header `Authorization` contra el `UserDetailsService` configurado. Si las credenciales son válidas, se autentica al usuario; de lo contrario, se retorna un 401 Unauthorized.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .httpBasic(Customizer.withDefaults())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        );

    return http.build();
}
```

```bash
# Ejemplo de peticion con HTTP Basic
curl -u usuario:password http://localhost:8080/api/productos

# O especificando el header manualmente
curl -H "Authorization: Basic dXN1YXJpbzpwYXNzd29yZA==" http://localhost:8080/api/productos
```

```java
// Configuracion completa para APIs REST
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .httpBasic(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/productos").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/productos").hasRole("ADMIN")
            .anyRequest().authenticated()
        );

    return http.build();
}
```

> HTTP Basic no es seguro sin HTTPS. El header `Authorization` contiene las credenciales codificadas en Base64, que es trivialmente decodificable. En producción, siempre se debe forzar HTTPS.

## Realm name

El realm name se envía en el header `WWW-Authenticate` y aparece en el diálogo nativo del navegador cuando se solicitan credenciales. Identifica el espacio de protección al que pertenecen las credenciales.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .httpBasic(basic -> basic
            .realmName("Mi API Rest")
        );

    return http.build();
}
```

```bash
# Respuesta del servidor sin autenticar
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Mi API Rest"
Content-Type: application/json

{"error": "Unauthorized", "message": "Credenciales requeridas"}
```

## Custom entry point

El `AuthenticationEntryPoint` personalizado controla qué respuesta se retorna cuando un cliente no autenticado intenta acceder a un recurso protegido. Por defecto, HTTP Basic retorna un 401 con el header `WWW-Authenticate`.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .httpBasic(basic -> basic
            .authenticationEntryPoint((request, response, authException) -> {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.setContentType("application/json;charset=UTF-8");
                response.getWriter().write("""
                    {
                        "error": "UNAUTHORIZED",
                        "message": "Credenciales invalidas o no proporcionadas",
                        "path": "%s"
                    }
                    """.formatted(request.getRequestURI()));
            })
        );

    return http.build();
}
```

```java
// Combinando HTTP Basic con OAuth2 en la misma aplicacion
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .httpBasic(Customizer.withDefaults())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        );

    return http.build();
}
```

HTTP Basic es un mecanismo de autenticación mínimo pero efectivo. Su simplicidad lo hace ideal para APIs internas y comunicación entre servicios donde no se necesita la complejidad de OAuth2 o JWT. La clave de su seguridad reside exclusivamente en la capa de transporte (HTTPS).

