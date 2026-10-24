# Auditing

El auditing permite registrar automáticamente quién modificó una entidad y cuándo, proporcionando un historial de cambios esencial para aplicaciones empresariales, sistemas financieros y cualquier contexto donde la trazabilidad sea obligatoria. Spring Data JPA integra un sistema de auditing que se ejecuta de forma transparente en el ciclo de vida de cada entidad persistente.

Sin un mecanismo automatizado de auditoría, los desarrolladores tienden a olvidar registrar campos de metadatos manualmente, lo que genera inconsistencias en los datos. La auditoría por framework garantiza que cada inserción, actualización y eliminación quede registrada de forma uniforme en toda la aplicación.

## Configuracion base

El auditing se habilita mediante `@EnableJpaAuditing` en la clase de configuración principal. Esta anotación activa el entity listener que intercepta las operaciones de persistencia y actualiza los campos de auditoría automáticamente.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Aplicacion {
    public static void main(String[] args) {
        SpringApplication.run(Aplicacion.class, args);
    }
}
```

> Si la aplicación define múltiples fuentes de datos, es posible que `@EnableJpaAuditing` cause conflictos. En ese caso, se puede deshabilitar la auditoría automática con `@EnableJpaAuditing(auditorAwareRef = "auditorProvider")` y proporcionar un bean personalizado.

## Entidad con campos de auditoria

Cada entidad que requiera auditoría debe incluir los campos de metadatos correspondientes, anotados con las anotaciones de auditing de Spring Data JPA.

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Documento {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String titulo;
    private String contenido;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime fechaCreacion;

    @LastModifiedDate
    private LocalDateTime fechaModificacion;

    @CreatedBy
    @Column(updatable = false)
    private String usuarioCreacion;

    @LastModifiedBy
    private String usuarioModificacion;

    // Constructores
    public Documento() {}

    public Documento(String titulo, String contenido) {
        this.titulo = titulo;
        this.contenido = contenido;
    }

    // Getters y setters
    public Long getId() { return id; }
    public String getTitulo() { return titulo; }
    public void setTitulo(String titulo) { this.titulo = titulo; }
    public String getContenido() { return contenido; }
    public void setContenido(String contenido) { this.contenido = contenido; }
    public LocalDateTime getFechaCreacion() { return fechaCreacion; }
    public LocalDateTime getFechaModificacion() { return fechaModificacion; }
    public String getUsuarioCreacion() { return usuarioCreacion; }
    public String getUsuarioModificacion() { return usuarioModificacion; }
}
```

`@EntityListeners(AuditingEntityListener.class)` vincula la entidad al listener de Spring Data que intercepta los eventos `@PrePersist`, `@PreUpdate` y `@PreRemove`. Sin esta anotación, los campos de auditoría no se actualizarán automáticamente.

## @CreatedDate y @LastModifiedDate

`@CreatedDate` se asigna una sola vez cuando la entidad se inserta por primera vez. `@LastModifiedDate` se actualiza en cada operación de persistencia posterior. Ambos campos deben ser de tipo `LocalDateTime`, `Instant` o `ZonedDateTime`.

```java
// Ejemplo de comportamiento
@Entity
public class Pedido {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime fechaCreacion; // Se establece una vez

    @LastModifiedDate
    private LocalDateTime fechaModificacion; // Se actualiza en cada save()
}
```

```java
@Service
public class PedidoService {

    public Pedido crearPedido(Pedido pedido) {
        pedido.setProducto("Laptop");
        pedido.setCantidad(2);
        return pedidoRepository.save(pedido);
        // fechaCreacion = 2024-01-15T10:30:00
        // fechaModificacion = 2024-01-15T10:30:00
    }

    public Pedido actualizarPedido(Pedido pedido) {
        pedido.setCantidad(5);
        return pedidoRepository.save(pedido);
        // fechaCreacion = 2024-01-15T10:30:00 (sin cambios)
        // fechaModificacion = 2024-01-15T14:45:00 (actualizado)
    }
}
```

> Los campos con `@CreatedDate` y `@CreatedBy` deben tener `updatable = false` en la anotación `@Column` para evitar que JPA intente actualizarlos en operaciones UPDATE.

## @CreatedBy y @LastModifiedBy

Estas anotaciones requieren un `AuditorAware<T>` que proporcione el identificador del usuario actual. Spring Data no determina el usuario autenticado por sí mismo; delega esa responsabilidad en el implementor.

```java
@Component
public class AuditorAwareImpl implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null || !authentication.isAuthenticated()) {
            return Optional.of("SYSTEM");
        }
        return Optional.of(authentication.getName());
    }
}
```

```java
// Con Spring Security y JWT
@Component
public class AuditorAwareJwt implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .filter(Authentication::isAuthenticated)
                .map(Authentication::getName);
    }
}
```

```java
// AuditorAware para multi-tenant
@Component
public class AuditorAwareMultiTenant implements AuditorAware<TenantUsuario> {

    private final TenantContext tenantContext;

    public AuditorAwareMultiTenant(TenantContext tenantContext) {
        this.tenantContext = tenantContext;
    }

    @Override
    public Optional<TenantUsuario> getCurrentAuditor() {
        return Optional.of(tenantContext.getUsuarioActual());
    }
}
```

## Configuracion avanzada

Para aplicaciones con múltiples fuentes de datos o configuraciones personalizadas, se puede crear un `JpaAuditingConfiguration` dedicado que controle el comportamiento de la auditoría.

```java
@Configuration
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
                .map(SecurityContext::getAuthentication)
                .filter(Authentication::isAuthenticated)
                .map(Authentication::getName)
                .or(() -> Optional.of("ANONYMOUS"));
    }
}
```

```java
// Entidad con campos de auditoria personalizados
@Entity
@EntityListeners(AuditingEntityListener.class)
public class AuditoriaCompleta {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(updatable = false)
    private Instant fechaCreacion;

    @LastModifiedDate
    private Instant fechaModificacion;

    @CreatedBy
    @Column(updatable = false)
    private String usuarioCreacion;

    @LastModifiedBy
    private String usuarioModificacion;

    // Campo personalizado: version para control de concurrencia optimista
    @Version
    private Long version;
}
```

> El auditing de Spring Data opera dentro de la misma transacción que la operación de persistencia. Si la transacción falla, los campos de auditoría no se persisten. Esto garantiza la consistencia entre los datos de negocio y los metadatos de auditoría.

