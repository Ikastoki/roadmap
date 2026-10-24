# CORS Configuration

Cross-Origin Resource Sharing (CORS) es un mecanismo de seguridad implementado por los navegadores que restringe las peticiones HTTP de origen cruzado. Cuando una aplicación web necesita acceder a recursos de un dominio diferente, el navegador bloquea la petición a menos que el servidor destino permita explícitamente esa originación. Spring MVC proporciona múltiples formas de configurar CORS para permitir peticiones cross-origin de manera controlada.

`@CrossOrigin` es la anotación más simple para habilitar CORS en controllers específicos o métodos individuales. Esta anotación define qué orígenes, métodos HTTP y headers están permitidos. Cuando se aplica a nivel de clase, la configuración se aplica a todos los métodos del controller. Cuando se aplica a un método específico, sobreescribe la configuración de clase.

```java
@RestController
@RequestMapping("/api/usuarios")
@CrossOrigin(origins = "http://localhost:3000")
public class UsuarioController {
    
    @GetMapping
    @CrossOrigin(origins = "*", methods = {RequestMethod.GET, RequestMethod.POST})
    public List<Usuario> listar() {
        return usuarioService.listar();
    }
    
    @GetMapping("/{id}")
    public Usuario obtener(@PathVariable Long id) {
        return usuarioService.obtenerPorId(id);
    }
    
    @PostMapping
    @CrossOrigin(origins = "http://localhost:3000", methods = RequestMethod.POST)
    public Usuario crear(@RequestBody @Valid Usuario usuario) {
        return usuarioService.crear(usuario);
    }
}
```

`WebMvcConfigurer.addCorsMappings` permite configurar CORS de manera global para toda la aplicación. Esta aproximación es preferida sobre `@CrossOrigin` cuando se necesita una política de CORS consistente. La configuración se define en una clase que implementa `WebMvcConfigurer` y establece reglas basadas en patrones de URL.

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000", "https://mi-app.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600)
            .exposedHeaders("X-Total-Count", "X-Total-Pages");
    }
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        // Configuración más permisiva para desarrollo
        registry.addMapping("/**")
            .allowedOrigins("*")
            .allowedMethods("*")
            .allowedHeaders("*");
    }
}
```

`CorsFilter` es un filtro de Servlet que se puede registrar como bean para manejar CORS a un nivel más bajo. Esta aproximación es útil cuando se necesita un control más granular sobre el procesamiento CORS, o cuando se integra con otros filtros de seguridad. El `CorsFilter` procesa las peticiones antes de que lleguen al `DispatcherServlet`.

```java
@Configuration
public class CorsFilterConfig {
    
    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("http://localhost:3000");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        config.addExposedHeader("X-Total-Count");
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        
        return new CorsFilter(source);
    }
}
```

La integración con Spring Security requiere una configuración CORS adicional, ya que Spring Security procesa las peticiones antes de que lleguen al MVC. Si se configura CORS solo en MVC, las peticiones preflight (OPTIONS) serán bloqueadas por Spring Security. La solución es habilitar CORS en la configuración de seguridad también.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList("http://localhost:3000"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        
        return source;
    }
}
```

> La configuración de CORS debe ser lo más restrictiva posible. Evitar `allowedOrigins("*")` con `allowCredentials(true)` ya que esto es una práctica insegura. Es mejor especificar los orígenes exactos que necesitan acceso y actualizar esta lista cuando sea necesario.

