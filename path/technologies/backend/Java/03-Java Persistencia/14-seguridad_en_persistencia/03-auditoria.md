# Auditoria

La auditoria de entidades consiste en registrar automaticamente quien creo un registro, cuando, quien lo modifico por ultima vez y en que momento. Esta informacion es esencial para cumplir con requisitos regulatorios (SOX, RGPD, PCI DSS), para trazar cambios en el sistema y para depurar incidencias de integridad de datos. Spring Data JPA y JPA 2.1 proporcionan mecanismos declarativos para implementar auditoria sin contaminar la logica de negocio.

## @CreatedDate, @LastModifiedDate, @CreatedBy, @LastModifiedBy

Spring Data JPA define cuatro anotaciones para marcar campos de auditoria en las entidades. Se activan mediante `@EntityListeners(AuditingEntityListener.class)` y la configuracion `@EnableJpaAuditing`.

```java
@Entity
@Table(name = "pedidos")
@EntityListeners(AuditingEntityListener.class)
public class Pedido {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String descripcion;
    private BigDecimal total;

    @CreatedDate
    @Column(name = "fecha_creacion", nullable = false, updatable = false)
    private LocalDateTime fechaCreacion;

    @LastModifiedDate
    @Column(name = "fecha_modificacion")
    private LocalDateTime fechaModificacion;

    @CreatedBy
    @Column(name = "creado_por", updatable = false)
    private String creadoPor;

    @LastModifiedBy
    @Column(name = "modificado_por")
    private String modificadoPor;
}
```

- `@CreatedDate`: establece la fecha/hora en el momento de la insercion. El campo se marca como `updatable = false` para que no se modifique en actualizaciones posteriores.
- `@LastModifiedDate`: se actualiza automaticamente en cada operacion de actualizacion.
- `@CreatedBy`: asigna el identificador del usuario que crea el registro. Depende de un `AuditorAware` que proporcione el usuario actual.
- `@LastModifiedBy`: se actualiza con cada modificacion.

## Habilitacion con @EnableJpaAuditing

La funcionalidad de auditoria de Spring Data JPA requiere activacion explicita mediante `@EnableJpaAuditing` en una clase de configuracion:

```java
@Configuration
@EnableJpaAuditing(
    auditorAwareRef = "auditorAware",
    dateTimeProviderRef = "dateTimeProvider",
    setDates = true,
    modifyOnCreate = true
)
public class AuditoriaConfig {

    @Bean
    public AuditorAware<String> auditorAware() {
        return () -> {
            // Obtener el usuario autenticado del contexto de seguridad
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth == null || !auth.isAuthenticated()) {
                return Optional.of("SYSTEM");
            }
            return Optional.of(auth.getName());
        };
    }

    @Bean
    public DateTimeProvider dateTimeProvider() {
        return () -> Optional.of(LocalDateTime.now());
    }
}
```

- `auditorAwareRef`: referencia al bean que implementa `AuditorAware`, encargado de devolver el identificador del usuario actual.
- `dateTimeProviderRef`: permite personalizar la fuente de fecha/hora (util para entornos clusterizados donde se necesita un reloj centralizado).
- `setDates`: si es `true` (valor por defecto), Spring Data establece `@CreatedDate` y `@LastModifiedDate` incluso si el campo ya tiene valor.
- `modifyOnCreate`: si es `true`, se establece `@LastModifiedBy` y `@LastModifiedDate` tambien en la creacion, no solo en actualizaciones.

## AuditorAware: proveedor del usuario actual

`AuditorAware<T>` es una interfaz funcional con un unico metodo `getCurrentAuditor()` que devuelve `Optional<T>`. El tipo generico `T` debe coincidir con el tipo de los campos `@CreatedBy` y `@LastModifiedBy`. Las implementaciones mas comunes obtienen el usuario del contexto de seguridad de Spring Security.

```java
@Component
public class SpringSecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName)
            .or(() -> Optional.of("ANONYMOUS"));
    }
}
```

Para sistemas que usan un identificador numerico de usuario:

```java
@Component
public class UsuarioIdAuditorAware implements AuditorAware<Long> {

    @Override
    public Optional<Long> getCurrentAuditor() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) return Optional.empty();
        Object principal = auth.getPrincipal();
        if (principal instanceof UserDetails) {
            Usuario usuario = (Usuario) ((UserDetails) principal).getCustomData();
            return Optional.of(usuario.getId());
        }
        return Optional.empty();
    }
}
```

## @EntityListeners(AuditingEntityListener.class)

`AuditingEntityListener` es la clase que Spring Data JPA registra como listener JPA. Intercepta los eventos `@PrePersist` y `@PreUpdate` para inyectar los valores de auditoria antes de que la entidad se persista o actualice. Sin esta anotacion en la entidad, `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy` y `@LastModifiedBy` no surten efecto.

```java
@EntityListeners(AuditingEntityListener.class)
public class Pedido { ... }
```

> Si varias entidades comparten los mismos campos de auditoria, se puede definir una clase base `@MappedSuperclass` con los campos y el `@EntityListeners`, y que las entidades concretas extiendan de ella. Esto evita repetir las anotaciones en cada entidad.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime fechaCreacion;

    @LastModifiedDate
    private LocalDateTime fechaModificacion;

    @CreatedBy
    @Column(updatable = false)
    private String creadoPor;

    @LastModifiedBy
    private String modificadoPor;
}

@Entity
@Table(name = "pedidos")
public class Pedido extends AuditableEntity {
    @Id private Long id;
    // resto de campos
}
```

## Auditoria manual con EventListeners de Hibernate

Si no se usa Spring Data, se puede implementar auditoria con los callbacks de JPA (`@PrePersist`, `@PreUpdate`, `@PreRemove`) directamente en la entidad o mediante un `EntityListener` externo.

```java
public class AuditoriaListener {

    @PrePersist
    public void antesDePersistir(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setFechaCreacion(LocalDateTime.now());
            auditable.setCreadoPor(obtenerUsuarioActual());
        }
    }

    @PreUpdate
    public void antesDeActualizar(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setFechaModificacion(LocalDateTime.now());
            auditable.setModificadoPor(obtenerUsuarioActual());
        }
    }

    private String obtenerUsuarioActual() {
        return SecurityContextHolder.getContext().getAuthentication().getName();
    }
}
```

La auditoria automatizada elimina el riesgo de olvidar rellenar estos campos manualmente y garantiza la trazabilidad completa de cada registro, desde su creacion hasta la ultima modificacion, sin importar que capa de la aplicacion realice la operacion.
