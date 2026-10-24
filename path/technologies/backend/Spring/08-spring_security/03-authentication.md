# Authentication

La autenticación en Spring Security se centra en la interfaz `UserDetailsService`, que define un único método `loadUserByUsername` responsable de cargar los datos del usuario desde cualquier fuente de almacenamiento. Spring Security no impone un modelo de datos específico; delegate la responsabilidad de buscar, validar y retornar un objeto `UserDetails` que contenga las credenciales, los roles y el estado de la cuenta.

El proceso de autenticación comienza cuando un filtro de autenticación (como `UsernamePasswordAuthenticationFilter` para formularios o `BasicAuthenticationFilter` para HTTP Basic) extrae las credenciales de la petición, las delega en un `AuthenticationManager`, que a su vez consulta al `UserDetailsService` y al `PasswordEncoder` para verificar que las credenciales sean válidas.

## UserDetailsService

`UserDetailsService` es la interfaz central que Spring Security utiliza para obtener la información del usuario durante la autenticación.

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

## Custom UserDetailsService

La implementación más habitual es consultar la base de datos mediante un repositorio JPA para cargar el usuario y sus roles.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UsuarioRepository usuarioRepository;

    public CustomUserDetailsService(UsuarioRepository usuarioRepository) {
        this.usuarioRepository = usuarioRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        Usuario usuario = usuarioRepository.findByEmail(email)
                .orElseThrow(() ->
                    new UsernameNotFoundException("Usuario no encontrado: " + email));

        return new CustomUserDetails(usuario);
    }
}
```

```java
@Entity
@Table(name = "usuarios")
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false)
    private boolean activo;

    @OneToMany(fetch = FetchType.EAGER)
    @JoinColumn(name = "usuario_id")
    private Set<Rol> roles;

    // Getters
    public String getEmail() { return email; }
    public String getPassword() { return password; }
    public boolean isActivo() { return activo; }
    public Set<Rol> getRoles() { return roles; }
}
```

```java
@Entity
@Table(name = "roles")
public class Rol {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String nombre; // Ej: "ROLE_ADMIN", "ROLE_USUARIO"

    public String getNombre() { return nombre; }
}
```

```java
public class CustomUserDetails implements UserDetails {

    private final Usuario usuario;

    public CustomUserDetails(Usuario usuario) {
        this.usuario = usuario;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return usuario.getRoles().stream()
                .map(rol -> new SimpleGrantedAuthority(rol.getNombre()))
                .collect(Collectors.toSet());
    }

    @Override
    public String getPassword() {
        return usuario.getPassword();
    }

    @Override
    public String getUsername() {
        return usuario.getEmail();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return usuario.isActivo();
    }
}
```

> Es importante que el nombre del campo en la petición de login coincida con lo que espera el `AuthenticationFilter`. Por defecto, espera `username` y `password`, pero se puede personalizar con `.usernameParameter("email")`.

## PasswordEncoder

`PasswordEncoder` es responsable de cifrar y verificar contraseñas. Spring Security nunca almacena contraseñas en texto plano; cada contraseña se cifra con un algoritmo de hash antes de persistirla.

```java
@Configuration
public class PasswordConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```java
@Service
public class RegistroService {

    private final UsuarioRepository usuarioRepository;
    private final PasswordEncoder passwordEncoder;

    public RegistroService(UsuarioRepository usuarioRepository,
                          PasswordEncoder passwordEncoder) {
        this.usuarioRepository = usuarioRepository;
        this.passwordEncoder = passwordEncoder;
    }

    public Usuario registrar(String email, String password) {
        if (usuarioRepository.findByEmail(email).isPresent()) {
            throw new IllegalArgumentException("El email ya esta registrado");
        }

        Usuario usuario = new Usuario();
        usuario.setEmail(email);
        usuario.setPassword(passwordEncoder.encode(password));
        usuario.setActivo(true);

        return usuarioRepository.save(usuario);
    }
}
```

> BCrypt genera un hash diferente cada vez que cifra la misma contraseña debido al salt aleatorio. Esto significa que dos usuarios con la misma contraseña tendrán hashes diferentes en la base de datos.

## AuthenticationManager

`AuthenticationManager` es el componente que coordina el proceso de autenticación. Recibe un objeto `Authentication` (que contiene las credenciales), delega en un `AuthenticationProvider` para verificar las credenciales y retorna un objeto `Authentication` autenticado o lanza una excepción.

```java
@Configuration
public class AuthConfig {

    @Bean
    public AuthenticationManager authenticationManager(
            HttpSecurity http,
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) throws Exception {

        AuthenticationManagerBuilder builder = http.getSharedObject(AuthenticationManagerBuilder.class);
        builder
            .userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder);

        return builder.build();
    }
}
```

## AuthenticationProvider

`AuthenticationProvider` es la interfaz que implementa la lógica específica de autenticación. Spring Security incluye `DaoAuthenticationProvider` que consulta un `UserDetailsService` y verifica las credenciales con un `PasswordEncoder`.

```java
@Bean
public AuthenticationProvider authenticationProvider(
        UserDetailsService userDetailsService,
        PasswordEncoder passwordEncoder) {

    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder);
    return provider;
}
```

```java
// Provider personalizado para autenticar con email o username
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    private final UsuarioRepository usuarioRepository;
    private final PasswordEncoder passwordEncoder;

    public CustomAuthenticationProvider(UsuarioRepository usuarioRepository,
                                       PasswordEncoder passwordEncoder) {
        this.usuarioRepository = usuarioRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {

        String entrada = authentication.getName();
        String password = authentication.getCredentials().toString();

        Usuario usuario = usuarioRepository.findByEmailOrUsername(entrada, entrada)
                .orElseThrow(() -> new BadCredentialsException("Credenciales invalidas"));

        if (!passwordEncoder.matches(password, usuario.getPassword())) {
            throw new BadCredentialsException("Credenciales invalidas");
        }

        if (!usuario.isActivo()) {
            throw new DisabledException("Cuenta deshabilitada");
        }

        List<SimpleGrantedAuthority> authorities = usuario.getRoles().stream()
                .map(rol -> new SimpleGrantedAuthority(rol.getNombre()))
                .collect(Collectors.toList());

        return new UsernamePasswordAuthenticationToken(usuario, password, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

## InMemoryUserDetailsManager

`InMemoryUserDetailsManager` almacena usuarios en memoria, útil para pruebas, prototipos o aplicaciones donde no se necesita persistencia de credenciales.

```java
@Configuration
public class InMemoryConfig {

    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder encoder) {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();

        UserDetails admin = User.builder()
                .username("admin")
                .password(encoder.encode("admin123"))
                .roles("ADMIN", "USUARIO")
                .build();

        UserDetails usuario = User.builder()
                .username("usuario")
                .password(encoder.encode("user123"))
                .roles("USUARIO")
                .build();

        manager.createUser(admin);
        manager.createUser(usuario);

        return manager;
    }
}
```

## JdbcUserDetailsManager

`JdbcUserDetailsManager` utiliza JDBC para almacenar y recuperar usuarios de una base de datos relacional, siguiendo un esquema predefinido de tablas `users` y `authorities`.

```java
@Configuration
public class JdbcConfig {

    @Bean
    public UserDetailsService userDetailsService(DataSource dataSource,
                                                 PasswordEncoder encoder) {
        JdbcUserDetailsManager manager = new JdbcUserDetailsManager(dataSource);

        // Crear usuario
        UserDetails usuario = User.builder()
                .username("nuevo")
                .password(encoder.encode("clave123"))
                .roles("USUARIO")
                .build();
        manager.createUser(usuario);

        return manager;
    }
}
```

> `JdbcUserDetailsManager` espera las tablas `users` (username, password, enabled) y `authorities` (username, authority) con el esquema por defecto de Spring Security. Se pueden personalizar las consultas con `setUsersByUsernameQuery` y `setAuthoritiesByUsernameQuery`.

La autenticación en Spring Security es un proceso modular donde cada componente tiene una responsabilidad clara. El `UserDetailsService` carga los datos, el `PasswordEncoder` verifica las credenciales y el `AuthenticationManager` coordina el proceso completo. Esta separación permite cambiar la fuente de usuarios o el mecanismo de cifrado sin modificar la lógica de autenticación de la aplicación.

