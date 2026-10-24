# Session Management

La gestión de sesiones en Spring Security controla cómo se crean, mantienen y destruyen las sesiones HTTP. Esto incluye políticas de creación (siempre, nunca, bajo demanda), protección contra fixation de sesiones, control de sesiones concurrentes y manejo de sesiones expiradas. Una mala gestión de sesiones puede exponer la aplicación a ataques de suplantación de identidad, fugas de memoria o denegación de servicio.

La sesión HTTP es el mecanismo por defecto que Spring Security utiliza para mantener el estado de autenticación del usuario. Cada sesión tiene un identificador único (JSESSIONID) que se almacena en una cookie del navegador. El servidor utiliza este identificador para recuperar el `SecurityContext` del usuario en cada petición subsiguiente.

## Session creation policy

`SessionCreationPolicy` controla cuándo Spring Security crea una sesión HTTP. Esta configuración es fundamental para determinar si la aplicación opera con estado (stateful) o sin estado (stateless).

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
        );

    return http.build();
}
```

| Politica                  | Comportamiento                                              |
|---------------------------|-------------------------------------------------------------|
| `IF_REQUIRED`             | Crea sesion solo si es necesaria (por defecto)              |
| `ALWAYS`                  | Siempre crea una sesion                                      |
| `NEVER`                   | No crea sesion, pero puede usar una existente                |
| `STATELESS`               | Nunca usa sesion, cada peticion es independiente             |

```java
// API REST stateless: sin sesiones
@Bean
public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        )
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

> `STATELESS` es la opción recomendada para APIs que utilizan JWT o autenticación basada en tokens. En este modo, Spring Security no almacena ni recupera `SecurityContext` de la sesión HTTP.

## Session fixation protection

El ataque de fixation de sesión ocurre cuando un atacante fuerza al usuario a utilizar un ID de sesión conocido. Si el usuario inicia sesión con ese ID, el atacante puede acceder a la cuenta usando el mismo ID. Spring Security protege contra esto regenerando el ID de sesión después de una autenticación exitosa.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .sessionFixation(fixation -> fixation
                .migrateSession() // Regenera ID y migra atributos (por defecto)
            )
        );

    return http.build();
}
```

| Estrategia            | Comportamiento                                                 |
|-----------------------|----------------------------------------------------------------|
| `migrateSession`      | Nuevo ID, copia atributos (por defecto)                        |
| `changeSessionId`     | Solo cambia el ID, mantiene atributos                          |
| `newSession`          | Crea sesion completamente nueva, sin atributos                 |
| `none`                | Sin proteccion (no recomendado)                                |

```java
// Proteccion maxima: nueva sesion sin atributos previos
.sessionManagement(session -> session
    .sessionFixation(fixation -> fixation
        .newSession()
    )
)
```

## Concurrent session control

El control de sesiones concurrentes limita cuántas sesiones activas puede tener un usuario simultáneamente. Esto previene que múltiples personas utilicen las mismas credenciales al mismo tiempo.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .maximumSessions(1)                    // Maximo 1 sesion por usuario
            .maxSessionsPreventsLogin(false)       // false: expulsa la sesion anterior
        );

    return http.build();
}
```

```java
// Si se intenta iniciar sesion con una cuenta ya activa
// maxSessionsPreventsLogin = false: la sesion anterior se invalida
// maxSessionsPreventsLogin = true: se bloquea el nuevo inicio de sesion

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .maximumSessions(2)
            .maxSessionsPreventsLogin(true) // Bloquea sesiones adicionales
            .expiredSessionStrategy(event -> {
                HttpServletResponse response = event.getResponse();
                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                response.setContentType("application/json");
                response.getWriter().write("{\"error\": \"Sesion expirada por limite concurrente\"}");
            })
        );

    return http.build();
}
```

```java
// Registrar el event listener para sesiones expiradas
@Bean
public SessionRegistry sessionRegistry() {
    return new SessionRegistryImpl();
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .sessionRegistry(sessionRegistry())
            .maximumSessions(1)
            .maxSessionsPreventsLogin(false)
        );

    return http.build();
}
```

## Expired sessions

Cuando una sesión expira o es invalidada, Spring Security puede ejecutar una estrategia personalizada para manejar el evento.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .invalidSessionUrl("/login?expired=true")
            .sessionConcurrency(concurrency -> concurrency
                .expiredSessionStrategy(event -> {
                    HttpServletResponse response = event.getResponse();
                    response.sendRedirect("/login?expired=true");
                })
            )
        );

    return http.build();
}
```

```java
// Configuracion de timeout de sesion en application.yml
server:
  servlet:
    session:
      timeout: 30m          # 30 minutos
      cookie:
        name: JSESSIONID
        http-only: true
        secure: true         # Solo HTTPS
        max-age: 1800        # 30 minutos en segundos
        same-site: lax       # Proteccion contra CSRF
```

```java
// Invalidar sesion al cerrar sesion manualmente
@GetMapping("/logout")
public String logout(HttpServletRequest request, HttpServletResponse response) {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth != null) {
        new SecurityContextLogoutHandler().logout(request, response, auth);
    }
    return "redirect:/login?logout=true";
}
```

La gestión de sesiones es un aspecto crítico de la seguridad de aplicaciones web. Configurar correctamente la creación de sesiones, la protección contra fixation y el control de concurrencia previene suplantación de identidad, uso indebido de credenciales y fugas de memoria por acumulación de sesiones inválidas.

