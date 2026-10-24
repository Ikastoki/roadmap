# Audit

Spring Boot Actuator proporciona un marco de trabajo para auditoria de eventos que permite registrar y consultar acciones significativas en la aplicacion. El sistema de auditoria se basa en `AuditEventRepository`, que almacena eventos de auditoria con un timestamp, principal (usuario), tipo de evento y datos adicionales. Esta funcionalidad es especialmente relevante en aplicaciones que manejan datos sensibles o que requieren trazabilidad regulatoria.

## AuditEventRepository

`AuditEventRepository` es la interfaz central del sistema de auditoria. Spring Boot autoconfigura una implementacion en memoria (`InMemoryAuditEventRepository`) que almacena los eventos en una lista circular con capacidad limitada.

```java
@Component
public class AuditoriaService {

    private final AuditEventRepository auditRepository;

    public AuditoriaService(AuditEventRepository auditRepository) {
        this.auditRepository = auditRepository;
    }

    public void registrarLogin(String usuario, boolean exitoso) {
        Map<String, Object> datos = new HashMap<>();
        datos.put("ip", obtenerIPActual());
        datos.put("user-agent", obtenerUserAgent());
        datos.put("exitoso", exitoso);

        auditRepository.add(new AuditEvent(
            usuario,
            exitoso ? "LOGIN_SUCCESS" : "LOGIN_FAILURE",
            datos
        ));
    }

    public void registrarCambio(String usuario, String entidad, String accion) {
        Map<String, Object> datos = new HashMap<>();
        datos.put("entidad", entidad);
        datos.put("accion", accion);
        datos.put("timestamp", Instant.now());

        auditRepository.add(new AuditEvent(usuario, "DATA_CHANGE", datos));
    }

    public List<AuditEvent> obtenerEventos(String usuario, Instant desde) {
        return auditRepository.find(usuario, desde, Instant.now());
    }
}
```

## @AuditListener

`@AuditListener` es un listener de JPA que registra automaticamente eventos de auditoria cada vez que se persiste, actualiza o elimina una entidad. Spring Boot detecta automaticamente este listener y lo registra en el `EntityManagerFactory`.

```java
@Entity
@EntityListeners(AuditListener.class)
public class Pedido {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private BigDecimal total;

    @Column(nullable = false)
    private String estado;

    @CreatedDate
    private Instant fechaCreacion;

    @LastModifiedDate
    private Instant fechaModificacion;

    @CreatedBy
    private String creadoPor;

    @LastModifiedBy
    private String modificadoPor;

    // Getters y setters
}
```

Para que `@CreatedBy` y `@LastModifiedBy` funcionen, se requiere un `AuditAware` que proporcione el usuario actual del contexto de seguridad.

```java
@Component
public class AuditoriaActual implements AuditAware {

    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

> `@AuditListener` solo funciona con JPA. Para aplicaciones que utilizan Spring Data Reactive (R2DBC, MongoDB), la auditoria debe implementarse de forma manual usando aspectos o handlers de eventos.

## Eventos de auditoria personalizados

Ademas de los eventos JPA automaticos, se pueden crear eventos de auditoria personalizados para registrar acciones de negocio especificas que requieren trazabilidad.

```java
@Component
public class AuditoriaAspect {

    private final AuditEventRepository auditRepository;
    private final AuthenticationFacade authFacade;

    public AuditoriaAspect(AuditEventRepository auditRepository,
                          AuthenticationFacade authFacade) {
        this.auditRepository = auditRepository;
        this.authFacade = authFacade;
    }

    @Around("@annotation(RegistrarAuditoria)")
    public Object auditar(ProceedingJoinPoint joinPoint) throws Throwable {
        String usuario = authFacade.getUsuarioActual();
        String metodo = joinPoint.getSignature().getName();
        String clase = joinPoint.getTarget().getClass().getSimpleName();

        Map<String, Object> datos = new HashMap<>();
        datos.put("metodo", metodo);
        datos.put("clase", clase);
        datos.put("argumentos", Arrays.toString(joinPoint.getArgs()));

        try {
            Object resultado = joinPoint.proceed();
            datos.put("resultado", "EXITOSO");
            auditRepository.add(new AuditEvent(usuario, "OPERACION_EXITOSA", datos));
            return resultado;
        } catch (Exception e) {
            datos.put("resultado", "FALLIDO");
            datos.put("error", e.getMessage());
            auditRepository.add(new AuditEvent(usuario, "OPERACION_FALLIDA", datos));
            throw e;
        }
    }
}
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RegistrarAuditoria {
}
```

Para entornos de produccion, `InMemoryAuditEventRepository` debe reemplazarse por una implementacion persistente que almacene los eventos en base de datos. Spring Data JPA ofrece `JpaAuditEventRepository` que persiste los eventos en una tabla `persistent_audit_event`.

```yaml
spring:
  data:
    jpa:
      repositories:
        audit:
          enabled: true
```

Los eventos de auditoria son fundamentales para el cumplimiento normativo (GDPR, SOX, PCI DSS) y para la resolucion de incidentes de seguridad. Un registro completo de quien hizo que, cuando y desde donde, permite reconstruir la secuencia de acciones en caso de brecha o comportamiento inesperado.

