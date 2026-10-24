# Que es Spring Security

Spring Security es un framework de seguridad declarativa que se integra profundamente en el ecosistema Spring para resolver los dos pilares fundamentales de la seguridad informática: autenticación (quién eres) y autorización (qué puedes hacer). Proporciona protección contra ataques comunes como CSRF, fixation, clickjacking y XSS, además de soporte completo para estándares como OAuth2, OIDC y SAML2. Cada petición HTTP que llega a una aplicación Spring pasa a través de una cadena de filtros de seguridad que inspectan, modifican o rechazan la solicitud antes de que llegue al controller.

La arquitectura de Spring Security se basa en el patrón de filtros del Servlet. No se trata de un único componente monolítico, sino de una cadena de filtros ordenada donde cada filtro tiene una responsabilidad específica: autenticar credenciales, validar tokens, verificar autorizaciones, gestionar sesiones o proteger contra ataques. Esta diseño modular permite reemplazar, agregar o eliminar filtros según las necesidades de la aplicación.

## Authentication vs Authorization

La **autenticación** verifica la identidad de un usuario. Cuando alguien intenta acceder al sistema, Spring Security intercepta la petición y determina si se trata de un usuario conocido. Esto puede hacerse mediante usuario/contraseña, tokens JWT, certificados TLS, OAuth2 o cualquier otro mecanismo de verificación de identidad.

La **autorización** determina si un usuario autenticado tiene permiso para acceder a un recurso o ejecutar una operación. Un usuario puede estar autenticado pero no tener permiso para acceder a ciertos endpoints o realizar ciertas acciones.

```java
// Ejemplo conceptual: un usuario esta autenticado pero no autorizado
// GET /api/admin/usuarios -> 403 Forbidden (autenticado sin rol ADMIN)
// GET /api/productos -> 200 OK (autenticado con cualquier rol)
```

## Arquitectura de seguridad

La cadena de filtros de seguridad está compuesta por múltiples `Filter` que se ejecutan en un orden específico. Cada filtro procesa la petición y decide si la deja pasar al siguiente filtro o la rechaza inmediatamente.

```
Peticion HTTP
    |
    v
SecurityFilterChain (cadena de filtros)
    |
    ├── SecurityContextPersistenceFilter (carga el contexto de seguridad)
    ├── HeaderWriterFilter (escribe headers de seguridad)
    ├── CsrfFilter (valida tokens CSRF)
    ├── AuthenticationFilter (autentica al usuario)
    ├── AuthorizationFilter (verifica autorizaciones)
    └── ... otros filtros
    |
    v
Controller (si la peticion es autorizada)
```

```java
// Configuracion basica de la cadena de filtros
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

> `SecurityFilterChain` reemplaza el antiguo `WebSecurityConfigurerAdapter` (deprecado desde Spring Security 6). La configuración funcional mediante `HttpSecurity` sigue el mismo principio pero con una API más limpia y orientada a lambdas.

## DelegatingFilterProxy

`DelegatingFilterProxy` es un servlet filter que delega la ejecución en un bean de Spring gestionado por el contenedor. Esto permite que el ciclo de vida de los filtros de seguridad esté controlado por Spring Framework en lugar del servlet container, habilitando la inyección de dependencias y la configuración declarativa.

```java
// Internamente, Spring Security registra este filtro en el Servlet container
// El proxy delega en el bean SecurityFilterChain del contexto de Spring
public class DelegatingFilterProxy extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        Filter delegateToBean = getDelegateBean();
        delegateToBean.doFilter(request, response, chain);
    }
}
```

```java
// En aplicaciones tradicionales, se registraba manualmente
// Spring Boot lo hace automaticamente
@Bean
public ServletRegistrationBean<DelegatingFilterProxy> securityFilterChain() {
    DelegatingFilterProxy proxy = new DelegatingFilterProxy("springSecurityFilterChain");
    return new ServletRegistrationBean<>(proxy, "/*");
}
```

Spring Boot detecta automáticamente la presencia de `spring-boot-starter-security` y aplica una configuración por defecto que protege todos los endpoints. Esta protección inicial se puede personalizar mediante la definición de un bean `SecurityFilterChain` que especifique las reglas de acceso específicas de la aplicación.

