# Async Support

Spring MVC proporciona soporte para operaciones asíncronas que permiten liberar el hilo del servlet mientras se realizan tareas de larga duración. Esto es fundamental para aplicaciones que necesitan manejar muchas conexiones concurrentes sin bloquear los recursos del servidor. El soporte asíncrono mejora significativamente la escalabilidad y el rendimiento de aplicaciones web que realizan operaciones como llamadas a APIs externas, procesamiento de datos o acceso a bases de datos.

`@EnableAsync` habilita el soporte asíncrono en la aplicación. Esta anotación se coloca en una clase de configuración o en la clase principal de la aplicación. Sin esta anotación, los métodos anotados con `@Async` se ejecutan de manera síncrona, anulando el propósito del soporte asíncrono.

`@Async` marca un método para ejecutarse en un hilo separado del hilo del servlet. Cuando se invoca un método anotado con `@Async`, Spring crea un nuevo hilo o utiliza un hilo de un pool configurado, y retorna inmediatamente al invocador. El método anotado se ejecuta en segundo plano mientras el hilo del servlet está disponible para manejar otras peticiones.

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.setThreadNamePrefix("AsyncThread-");
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificacionService {
    
    @Async("taskExecutor")
    public void enviarEmailAsincrono(String destino, String asunto, String contenido) {
        // Esta tarea se ejecuta en un hilo separado
        emailService.enviar(destino, asunto, contenido);
    }
    
    @Async
    public CompletableFuture<Reporte> generarReporte(Long usuarioId) {
        // Retorna un CompletableFuture para manejo asíncrono de resultados
        Reporte reporte = reporteService.generar(usuarioId);
        return CompletableFuture.completedFuture(reporte);
    }
    
    @Async
    public void procesarDatos(List<Dato> datos) {
        // Procesamiento pesado en segundo plano
        datos.forEach(dato -> {
            // Procesamiento individual
            procesador.procesar(dato);
        });
    }
}
```

`DeferredResult` permite retornar un resultado que se completará de manera asíncrona. A diferencia de `Callable`, `DeferredResult` no está atado a un hilo del container de servlets. Esto permite que el resultado se establezca desde cualquier hilo, incluso después de que el hilo del servlet haya sido liberado. `DeferredResult` es ideal para escenarios donde el resultado depende de un evento externo.

```java
@RestController
@RequestMapping("/api/async")
public class DeferredResultController {
    
    private final Map<Long, DeferredResult<String>> resultados = new ConcurrentHashMap<>();
    
    @GetMapping("/deferred/{id}")
    public DeferredResult<String> obtenerDeferred(@PathVariable Long id) {
        DeferredResult<String> result = new DeferredResult<>(Duration.ofSeconds(30).toMillis());
        
        resultados.put(id, result);
        
        result.onTimeout(() -> {
            resultados.remove(id);
            result.setErrorResult("Timeout");
        });
        
        result.onCompletion(() -> resultados.remove(id));
        
        return result;
    }
    
    // Este método simula un evento que completa el DeferredResult
    public void completarResultado(Long id, String valor) {
        DeferredResult<String> result = resultados.remove(id);
        if (result != null) {
            result.setResult(valor);
        }
    }
}
```

`Callable` es una forma simple de ejecutar una tarea asíncrona dentro del contexto del servlet. El método retorna un `Callable` que se ejecuta en un hilo del pool de threads del container. A diferencia de `DeferredResult`, `Callable` está limitado por los recursos del container de servlets.

```java
@RestController
@RequestMapping("/api/callable")
public class CallableController {
    
    @GetMapping("/tarea")
    public Callable<ResponseEntity<String>> ejecutarTareaAsincrona() {
        return () -> {
            // Simular trabajo pesado
            Thread.sleep(2000);
            String resultado = "Tarea completada";
            return ResponseEntity.ok(resultado);
        };
    }
    
    @GetMapping("/datos/{id}")
    public Callable<ResponseEntity<Datos>> obtenerDatos(@PathVariable Long id) {
        return () -> {
            Datos datos = servicioExterno.obtenerDatos(id);
            return ResponseEntity.ok(datos);
        };
    }
}
```

`WebAsyncTask` ofrece más control sobre la ejecución asíncrona que `Callable`. Permite configurar el timeout, el executor personalizado y callbacks de timeout y completación. Esta opción es útil cuando se necesita un control granular sobre el comportamiento asíncrono.

```java
@RestController
@RequestMapping("/api/web-async")
public class WebAsyncTaskController {
    
    @GetMapping("/tarea/{id}")
    public WebAsyncTask<ResponseEntity<String>> ejecutarConTimeout(@PathVariable Long id) {
        
        WebAsyncTask<ResponseEntity<String>> asyncTask = new WebAsyncTask<>(10000L, () -> {
            // Tarea que puede tardar
            String resultado = servicioProcesamiento.procesar(id);
            return ResponseEntity.ok(resultado);
        });
        
        asyncTask.onTimeout(() -> {
            return ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT)
                .body("Tiempo de espera excedido");
        });
        
        asyncTask.onCompletion(() -> {
            // Callback cuando la tarea termina (éxito o timeout)
            log.info("Tarea completada para id: {}", id);
        });
        
        return asyncTask;
    }
}
```

Server-Sent Events (SSE) permiten establecer una conexión persistente del servidor al cliente para enviar actualizaciones en tiempo real. A diferencia de WebSockets, SSE es unidireccional (solo del servidor al cliente) y funciona sobre HTTP estándar. Spring MVC soporta SSE mediante el retorno de `SseEmitter` o `Flux<ServerSentEvent>`.

```java
@RestController
@RequestMapping("/api/sse")
public class SseController {
    
    private final List<SseEmitter> emitters = new CopyOnWriteArrayList<>();
    
    @GetMapping("/events")
    public SseEmitter conectarse() {
        SseEmitter emitter = new SseEmitter(Duration.ofMinutes(5).toMillis());
        
        emitters.add(emitter);
        
        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));
        emitter.onError(e -> emitters.remove(emitter));
        
        // Enviar evento de conexión
        try {
            emitter.send(SseEmitter.event()
                .name("conexion")
                .data("Conectado al servidor de eventos"));
        } catch (IOException e) {
            emitters.remove(emitter);
        }
        
        return emitter;
    }
    
    public void enviarEvento(String nombre, Object datos) {
        List<SseEmitter> emittersCopy = new ArrayList<>(emitters);
        for (SseEmitter emitter : emittersCopy) {
            try {
                emitter.send(SseEmitter.event()
                    .name(nombre)
                    .data(datos));
            } catch (IOException e) {
                emitters.remove(emitter);
            }
        }
    }
}
```

> El soporte asíncrono no debe usarse indiscriminadamente. Las operaciones rápidas no se benefician del async overhead. Es ideal para operaciones de larga duración como llamadas a APIs externas, procesamiento de archivos o consultas complejas que bloquearían el hilo del servlet durante mucho tiempo.

