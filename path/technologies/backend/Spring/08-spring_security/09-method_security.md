# Method Security

La seguridad basada en métodos protege la lógica de negocio directamente en las clases de servicio, sin depender exclusivamente de la configuración de URL. Mientras que `authorizeHttpRequests` actúa como una barrera externa (quién puede acceder a qué URL), la seguridad de métodos se infiltra en el código mismo, verificando permisos antes o después de ejecutar cada método. Esto es especialmente valioso cuando diferentes partes de la aplicación necesitan distintos niveles de acceso a la misma entidad.

La seguridad de métodos es la segunda línea de defensa. Si un atacante encuentra una forma de evadir la configuración de URL (por ejemplo, a través de una vulnerabilidad en el routing), las anotaciones en los métodos siguen protegiendo la lógica de negocio.

## @EnableMethodSecurity

`@EnableMethodSecurity` habilita el procesamiento de anotaciones de seguridad en métodos de Spring. Sin esta anotación, las anotaciones como `@PreAuthorize` y `@Secured` son ignoradas.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(
    prePostEnabled = true,   // Habilita @PreAuthorize y @PostAuthorize
    securedEnabled = true,   // Habilita @Secured
    jsr250Enabled = true     // Habilita @RolesAllowed
)
public class SecurityConfig {
}
```

## @PreAuthorize

`@PreAuthorize` evalúa una expresión SpEL **antes** de la ejecución del método. Si la expresión retorna `false`, se lanza `AccessDeniedException` y el método nunca se ejecuta.

```java
@Service
@Transactional
public class UsuarioService {

    private final UsuarioRepository usuarioRepository;
    private final PasswordEncoder passwordEncoder;

    public UsuarioService(UsuarioRepository usuarioRepository,
                         PasswordEncoder passwordEncoder) {
        this.usuarioRepository = usuarioRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @PreAuthorize("hasRole('ADMIN')")
    public List<Usuario> listarTodos() {
        return usuarioRepository.findAll();
    }

    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public Usuario obtenerPorId(Long id) {
        return usuarioRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Usuario no encontrado"));
    }

    @PreAuthorize("hasRole('ADMIN') or #dto.email == authentication.name")
    public Usuario actualizar(Long id, UsuarioDTO dto) {
        Usuario usuario = usuarioRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Usuario no encontrado"));
        usuario.setNombre(dto.getNombre());
        return usuarioRepository.save(usuario);
    }

    @PreAuthorize("hasRole('ADMIN')")
    public void eliminar(Long id) {
        usuarioRepository.deleteById(id);
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'MODERADOR')")
    public void cambiarEstado(Long id, boolean activo) {
        Usuario usuario = usuarioRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Usuario no encontrado"));
        usuario.setActivo(activo);
        usuarioRepository.save(usuario);
    }
}
```

## @PostAuthorize

`@PostAuthorize` evalúa una expresión **después** de la ejecución del método. La variable `returnObject` representa el valor retornado por el método. Esto permite decisiones de autorización basadas en el contenido del resultado.

```java
@Service
public class DocumentoService {

    private final DocumentoRepository documentoRepository;

    public DocumentoService(DocumentoRepository documentoRepository) {
        this.documentoRepository = documentoRepository;
    }

    @PostAuthorize("returnObject.autor.email == authentication.name or hasRole('ADMIN')")
    public Documento obtenerPorId(Long id) {
        return documentoRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Documento no encontrado"));
    }

    @PostAuthorize("returnObject.vendedor.id == authentication.principal.id or hasRole('ADMIN')")
    public Venta obtenerVenta(Long id) {
        return ventaRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Venta no encontrada"));
    }

    @PostFilter("filterObject.publico == true or filterObject.autor.email == authentication.name")
    public List<Documento> listarDocumentos() {
        return documentoRepository.findAll();
    }

    @PreFilter("filterObject.autor.email == authentication.name")
    public void eliminarDocumentos(Collection<Documento> documentos) {
        documentoRepository.deleteAll(documentos);
    }
}
```

> `@PostFilter` y `@PreFilter` filtran colecciones. En `@PreFilter`, solo se procesan los elementos donde la expresión retorna `true`. En `@PostFilter`, se eliminan del resultado los elementos que no cumplen la condición.

## @Secured

`@Secured` es una anotación simple que verifica si el usuario tiene al menos uno de los roles especificados. No soporta expresiones SpEL.

```java
@Secured("ROLE_ADMIN")
public void configurarSistema() {
    // Solo accesible por admin
}

@Secured({"ROLE_ADMIN", "ROLE_USUARIO"})
public List<Producto> verProductos() {
    // Accesible por admin y usuario
}
```

## @RolesAllowed (JSR-250)

`@RolesAllowed` es una anotación estándar Java EE que ofrece la misma funcionalidad que `@Secured` pero con soporte portátil.

```java
@RolesAllowed("ADMIN")
public void gestionarReportes() {
    // ...
}

@RolesAllowed({"ADMIN", "AUDITOR"})
public void auditarOperaciones() {
    // ...
}
```

## SpEL en seguridad de metodos

Las expresiones SpEL proporcionan flexibilidad total para definir reglas de autorización complejas.

```java
@Service
public class ProyectoService {

    @PreAuthorize("#proyecto.empresa.id == authentication.principal.empresaId")
    public void editarProyecto(Proyecto proyecto) {
        // Solo miembros de la misma empresa pueden editar
    }

    @PreAuthorize("hasRole('ADMIN') or (hasRole('EDITOR') and #dto.categoria == 'INTERNO')")
    public void crearDocumento(DocumentoDTO dto) {
        // Admin puede crear cualquier documento, editor solo internos
    }

    @PostAuthorize("returnObject.monto <= authentication.principal.limiteCredito")
    public Venta crearVenta(VentaDTO dto) {
        // El monto no puede exceder el limite de credito del usuario
    }

    @PreAuthorize("@securityService.isOwner(#id, authentication.name)")
    public void actualizarRecurso(Long id, RecursoDTO dto) {
        // Delega la verificacion a un servicio externo
    }
}
```

```java
@Component("securityService")
public class SecurityService {

    private final RecursoRepository recursoRepository;

    public SecurityService(RecursoRepository recursoRepository) {
        this.recursoRepository = recursoRepository;
    }

    public boolean isOwner(Long recursoId, String username) {
        return recursoRepository.findById(recursoId)
                .map(recurso -> recurso.getPropietario().getEmail().equals(username))
                .orElse(false);
    }

    public boolean isMemberOf(Long empresaId, String username) {
        return usuarioRepository.existsByEmailAndEmpresaId(username, empresaId);
    }
}
```

> En expresiones SpEL, `@` permite invocar beans del contexto de Spring. Esto es útil para lógica de autorización compleja que no se puede expresar inline.

La seguridad basada en métodos proporciona una defensa en profundidad que complementa la autorización a nivel de URL. Mientras que `authorizeHttpRequests` protege los endpoints de acceso no autorizado, las anotaciones de método protegen la lógica de negocio internamente, asegurando que incluso si un código malicioso logra invocar un método directamente, las verificaciones de permisos se siguen ejecutando.

