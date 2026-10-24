# Authorization

La autorización en Spring Security determina si un usuario autenticado tiene permiso para acceder a un recurso o ejecutar una operación. Existen dos niveles principales de autorización: a nivel de URL (basada en la ruta de la petición) y a nivel de método (basada en la ejecución de código). Ambos mecanismos pueden coexistir en la misma aplicación, ofreciendo una defensa en profundidad donde las reglas de URL actúan como primera barrera y las anotaciones de método como segunda capa.

La autorización se evalúa después de la autenticación. Si el usuario no está autenticado, Spring Security redirige al mecanismo de login correspondiente antes de evaluar cualquier regla de autorización. Una vez autenticado, el objeto `Authentication` se almacena en el `SecurityContext` y está disponible para todas las decisiones de acceso.

## requestMatchers

`requestMatchers` es el método principal para definir reglas de autorización basadas en URL. Patrones de URL y condiciones de acceso se combinan de forma declarativa en la configuración del `SecurityFilterChain`.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/").permitAll()
            .requestMatchers("/home").permitAll()
            .requestMatchers("/login").permitAll()
            .requestMatchers("/api/public/**").permitAll()

            .requestMatchers(HttpMethod.GET, "/api/productos").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/productos").hasRole("ADMIN")
            .requestMatchers(HttpMethod.PUT, "/api/productos/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.DELETE, "/api/productos/**").hasRole("ADMIN")

            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .requestMatchers("/api/usuarios/**").hasAnyRole("ADMIN", "GERENTE")
            .requestMatchers("/api/reportes/**").hasAnyAuthority("REPORTES", "ADMIN")

            .anyRequest().authenticated()
        );

    return http.build();
}
```

## hasRole

`hasRole` verifica que el usuario tenga un rol específico. Spring Security añade automáticamente el prefijo `ROLE_` al nombre del rol, por lo que `hasRole("ADMIN")` busca el authorities `ROLE_ADMIN`.

```java
// En la configuracion
.requestMatchers("/api/admin/**").hasRole("ADMIN")

// Equivalente a buscar el authority "ROLE_ADMIN" en el Authentication
// El prefijo ROLE_ se anade automaticamente
```

```java
// hasAnyRole: verifica multiples roles
.requestMatchers("/api/usuarios/**").hasAnyRole("ADMIN", "MODERADOR")
```

```java
// hasAuthority: busca el authority exacto sin prefijo automatico
.requestMatchers("/api/reportes/**").hasAuthority("REPORTES_VER")
.requestMatchers("/api/config/**").hasAuthority("CONFIGURACION_ESCRIBIR")
```

> La diferencia entre `hasRole` y `hasAuthority` es que `hasRole` añade el prefijo `ROLE_` automáticamente, mientras que `hasAuthority` busca el nombre exacto. Si los roles en la base de datos ya incluyen el prefijo `ROLE_`, se debe usar `hasAuthority` para evitar duplicación.

## authenticated y permitAll

`authenticated` requiere que el usuario esté autenticado pero no especifica ningún rol o authority particular. `permitAll` permite el acceso sin autenticación, útil para endpoints públicos.

```java
.requestMatchers("/api/productos").authenticated()  // Requiere login
.requestMatchers("/api/public/**").permitAll()       // Sin restricciones
```

```java
// authenticated es el comportamiento por defecto de anyRequest
.anyRequest().authenticated()
```

## SpEL en autorizacion

Spring Security soporta expresiones SpEL (Spring Expression Language) para definir reglas de autorización complejas que no se pueden expresar con los métodos estándar.

```java
.requestMatchers("/api/usuarios/{id}/perfil")
    .access(new SpelAuthorizationManager("""
        @authenticationPrincipal?.username == #id
        or hasRole('ADMIN')
    """))

.requestMatchers("/api/empresas/{empresaId}/datos")
    .access(new SpelAuthorizationManager("""
        hasRole('ADMIN')
        or (
            @employeeService.isEmployeeOfCompany(
                @authenticationPrincipal?.username,
                #empresaId
            )
            and hasAuthority('EMPRESA_LEER')
        )
    """))
```

## @PreAuthorize

`@PreAuthorize` evalúa una expresión SpEL **antes** de la ejecución del método. Si la expresión retorna `false`, se lanza `AccessDeniedException`.

```java
@Service
public class VentaService {

    @PreAuthorize("hasRole('ADMIN')")
    public void eliminarVenta(Long id) {
        ventaRepository.deleteById(id);
    }

    @PreAuthorize("hasRole('ADMIN') or @ventaRepository.findById(#id).get().vendedor.username == authentication.name")
    public Venta obtenerVenta(Long id) {
        return ventaRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Venta no encontrada"));
    }

    @PreAuthorize("#producto.precio > 0 and #producto.precio < 10000")
    public void validarProducto(Producto producto) {
        // Solo permite productos con precio entre 0 y 10000
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'VENDEDOR') and #dto.monto <= 50000")
    public Venta crearVenta(VentaDTO dto) {
        // Solo vendedores y admin pueden crear ventas de hasta 50000
    }
}
```

## @PostAuthorize

`@PostAuthorize` evalúa una expresión **después** de la ejecución del método, accediendo al resultado mediante la variable `returnObject`. Es útil cuando la decisión de autorización depende del contenido del objeto retornado.

```java
@Service
public class DocumentoService {

    @PostAuthorize("returnObject.autor.username == authentication.name or hasRole('ADMIN')")
    public Documento obtenerDocumento(Long id) {
        return documentoRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Documento no encontrado"));
    }

    @PostAuthorize("returnObject.size() <= authentication.principal.maxDocumentos")
    public List<Documento> listarDocumentos() {
        return documentoRepository.findAll();
    }

    @PostFilter("filterObject.publico == true or filterObject.autor.username == authentication.name")
    public List<Documento> listarTodosLosDocumentos() {
        return documentoRepository.findAll();
    }

    @PreFilter("filterObject.autor.username == authentication.name")
    public void eliminarDocumentos(List<Documento> documentos) {
        documentoRepository.deleteAllInBatch(documentos);
    }
}
```

> `@PostFilter` y `@PreFilter` filtran colecciones antes o después de la ejecución del método. `filterObject` representa cada elemento de la colección en la evaluación de la expresión.

## @Secured

`@Secured` es la forma más simple de autorización basada en anotaciones. Solo acepta una lista de roles y no soporta expresiones SpEL.

```java
@Secured("ROLE_ADMIN")
public void eliminarUsuario(Long id) {
    usuarioRepository.deleteById(id);
}

@Secured({"ROLE_ADMIN", "ROLE_MODERADOR"})
public void moderarContenido(Long id) {
    // ...
}
```

> `@Secured` no soporta expresiones SpEL. Para reglas complejas, se debe usar `@PreAuthorize` o `@PostAuthorize`.

## @RolesAllowed (JSR-250)

`@RolesAllowed` es una anotación estándar de Java EE (JSR-250) que ofrece funcionalidad similar a `@Secured` pero con soporte portátil entre diferentes proveedores de seguridad.

```java
@RolesAllowed("ADMIN")
public void configurarSistema() {
    // ...
}

@RolesAllowed({"ADMIN", "USUARIO"})
public void verPerfil() {
    // ...
}
```

```java
// Habilitar @RolesAllowed en Spring Security
@Configuration
@EnableMethodSecurity(jsr250Enabled = true)
public class SecurityConfig {
}
```

## Role Hierarchy

La jerarquía de roles permite definir relaciones de herencia entre roles. Un usuario con un rol superior hereda automáticamente los permisos de los roles inferiores.

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.withRolePrefix("ROLE_")
            .role("ADMIN").implies("MODERADOR")
            .role("MODERADOR").implies("USUARIO")
            .role("USUARIO").implies("ANONIMO")
            .build();
}
```

```yaml
# Alternativa: definicion en archivo
# role-hierarchy.yml
roleHierarchy:
    ROLE_ADMIN > ROLE_MODERADOR
    ROLE_MODERADOR > ROLE_USUARIO
    ROLE_USUARIO > ROLE_ANONIMO
```

Con esta configuración, un usuario con `ROLE_ADMIN` tiene implícitamente acceso a todo lo que requiere `ROLE_MODERADOR`, `ROLE_USUARIO` y `ROLE_ANONIMO`. Esto simplifica las reglas de autorización y evita la duplicación de permisos.

```java
// Con jerarquia de roles, esto funciona automaticamente
.requestMatchers("/api/admin/**").hasRole("ADMIN")      // Solo admin
.requestMatchers("/api/moderar/**").hasRole("MODERADOR") // Admin y moderador
.requestMatchers("/api/usuarios/**").hasRole("USUARIO")  // Admin, moderador y usuario
```

La autorización en Spring Security ofrece múltiples niveles de granularidad. Desde la configuración de URL que actúa como barrera inicial, hasta las anotaciones de método que protegen la lógica de negocio específica. Combinar ambos enfoques con una jerarquía de roles bien definida proporciona una seguridad robusta y mantenible.

