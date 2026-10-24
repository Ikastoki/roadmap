# Custom Endpoints

Spring Boot Actuator permite crear endpoints personalizados que exponen informacion o acciones especificas de la aplicacion. Los endpoints customizados siguen el mismo mecanismo de exposicion y seguridad que los endpoints estandar, pero implementan la logica que el desarrollador considere necesaria. Son la forma de extender Actuator con funcionalidad de monitorizacion y administracion especifica del dominio de la aplicacion.

## Anotacion @Endpoint

Un endpoint personalizado se define como una clase anotada con `@Endpoint`. Cada metodo dentro de la clase puede exponerse como una operacion de lectura, escritura o eliminacion mediante las anotaciones correspondientes.

```java
@Component
@Endpoint(id = "cache")
public class CacheEndpoint {

    private final CacheManager cacheManager;

    public CacheEndpoint(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @ReadOperation
    public Map<String, Object> estadisticas() {
        Map<String, Object> stats = new HashMap<>();
        for (String nombre : cacheManager.getCacheNames()) {
            Cache cache = cacheManager.getCache(nombre);
            if (cache instanceof EhCacheCache ehCache) {
                stats.put(nombre, Map.of(
                    "entries", ehCache.getNativeCache().getSize(),
                    "hits", ehCache.getNativeCache().getStatistics()
                        .getCacheHits(),
                    "misses", ehCache.getNativeCache().getStatistics()
                        .getCacheMisses()
                ));
            }
        }
        return stats;
    }

    @WriteOperation
    public Map<String, String> limpiarCache(
            @Selector String nombreCache) {
        Cache cache = cacheManager.getCache(nombreCache);
        if (cache != null) {
            cache.clear();
            return Map.of("status", "ok", "cache", nombreCache);
        }
        return Map.of("status", "not_found", "cache", nombreCache);
    }

    @DeleteOperation
    public Map<String, String> eliminarCache(
            @Selector String nombreCache) {
        cacheManager.getCacheNames()
            .stream()
            .filter(name -> name.equals(nombreCache))
            .forEach(name -> {
                Cache cache = cacheManager.getCache(name);
                if (cache != null) cache.clear();
            });
        return Map.of("status", "deleted", "cache", nombreCache);
    }
}
```

## @ReadOperation, @WriteOperation, @DeleteOperation

Cada anotacion de operacion determina el metodo HTTP que la expone:

- `@ReadOperation` se mapea a `GET` y no debe producir efectos colaterales en el sistema
- `@WriteOperation` se mapea a `POST` y puede modificar el estado de la aplicacion
- `@DeleteOperation` se mapea a `DELETE` y puede eliminar recursos

```java
@Component
@Endpoint(id = "scheduled-tasks")
public class ScheduledTasksEndpoint {

    private final ThreadPoolTaskScheduler scheduler;

    public ScheduledTasksEndpoint(ThreadPoolTaskScheduler scheduler) {
        this.scheduler = scheduler;
    }

    @ReadOperation
    public Map<String, Object> tareasActivas() {
        return Map.of(
            "poolSize", scheduler.getPoolSize(),
            "activeCount", scheduler.getActiveCount(),
            "queueSize", scheduler.getThreadPoolExecutor()
                .getQueue().size()
        );
    }

    @WriteOperation
    public Map<String, String> reiniciarScheduler() {
        scheduler.shutdown();
        scheduler.initialize();
        return Map.of("status", "reiniciado");
    }
}
```

El parametro `@Selector` permite que el endpoint acepte un segmento de la URL como argumento, creando sub-recursos bajo el path del endpoint. Por ejemplo, `@Endpoint(id = "cache")` con `@Selector` genera endpoints como `/actuator/cache/{nombreCache}`.

## Configuracion de exposicion

La exposicion de endpoints personalizados se configura en `application.yml` de la misma forma que los endpoints estandar. Solo los endpoints que se incluyan explicitamente en la configuracion seran accesibles.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,cache,scheduled-tasks
      base-path: /actuator
  endpoint:
    cache:
      enabled: true
      show-details: when_authorized
      sensitive: true
    scheduled-tasks:
      enabled: true
      sensitive: false
```

> Los endpoints de escritura y eliminacion son inherentemente peligrosos en produccion. Se recomienda restringir su exposicion con autenticacion y autorizacion estricta, o limitarlos a perfiles de desarrollo.

## Endpoints JMX

Ademas de HTTP, los endpoints personalizados pueden exponerse como beans MBean en JMX (Java Management Extensions). Esto permite integrar la monitorizacion con herramientas de gestion empresariales como JConsole o VisualVM.

```yaml
management:
  endpoints:
    jmx:
      exposure:
        include: health,metrics,cache
  endpoint:
    cache:
      jmx:
        enabled: true
```

La exposicion JMX es particularmente util en entornos empresariales donde las herramientas de monitorizacion existentes se comunican via JMX en lugar de HTTP. Spring Boot gestiona automaticamente la creacion de MBeans para cada operacion expuesta.

La combinacion de `@ReadOperation`, `@WriteOperation` y `@DeleteOperation` permite construir una API de administracion completa para cualquier componente de la aplicacion: colas de mensajes, pools de conexiones, caches, colas de procesamiento o cualquier recurso que necesite ser monitoreado o gestionado remotamente.

