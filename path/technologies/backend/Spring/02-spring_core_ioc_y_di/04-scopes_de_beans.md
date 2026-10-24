# Scopes de beans

El scope de un bean define el ciclo de vida y la visibilidad de una instancia dentro del contenedor IoC de Spring. Determina cuantas instancias del bean se crean y en que contexto estan disponibles. La eleccion del scope adecuado es critica para el correcto funcionamiento de la aplicacion: un bean con un scope inadecuado puede causar fugas de memoria, condiciones de carrera, comportamiento inconsistente entre peticiones o errores de concurrencia.

Spring define seis scopes incorporados, cada uno adaptado a un caso de uso especifico. Ademas, permite la creacion de scopes personalizados que se integran en el ciclo de vida normal del contenedor.

## Singleton: el scope por defecto

El scope `singleton` indica que el contenedor crea una unica instancia del bean y la reutiliza para todas las inyecciones y peticiones a lo largo de la vida de la aplicacion. Este es el scope por defecto y el mas comun. Un bean singleton se instancia una vez durante la inicializacion del contenedor (eager loading) o bajo demanda (lazy loading), y permanece en memoria hasta que el contenedor se destruye.

```java
@Component
@Scope("singleton") // Equivalente a no especificar @Scope
public class ServicioConfiguracion {

    private final Map<String, String> propiedades = new ConcurrentHashMap<>();

    @PostConstruct
    public void cargarPropiedades() {
        propiedades.put("modo", "produccion");
        propiedades.put("version", "2.1.0");
    }

    public String obtener(String clave) {
        return propiedades.get(clave);
    }
}
```

Los beans singleton son ideales para servicios sin estado, repositorios, componentes de infraestructura y cualquier objeto que pueda ser compartido de forma segura entre multiples hilos. Sin embargo, no son apropiados para beans que mantienen estado mutable que cambia entre peticiones, ya que todos los usuarios comparten la misma instancia.

> Un bean singleton no es lo mismo que el patron Singleton de GoF. El patron Singleton controla cuantas instancias existen en la JVM, mientras que el scope singleton de Spring controla cuantas instancias existen dentro de un contenedor. En una aplicacion con multiples contextos de Spring (por ejemplo, un contexto raiz y un contexto servlet), puede existir mas de una instancia del mismo bean singleton.

## Prototype: nueva instancia por cada inyeccion

El scope `prototype` indica que el contenedor crea una nueva instancia del bean cada vez que se solicita via `getBean()` o se inyecta en otro bean. A diferencia de los beans singleton, el contenedor no gestiona el ciclo de vida completo de los beans prototype: crea la instancia, realiza la inyeccion de dependencias y ejecuta los callbacks de inicializacion, pero no ejecuta los callbacks de destruccion.

```java
@Component
@Scope("prototype")
public class CarritoCompra {

    private final List<ItemCarrito> items = new ArrayList<>();
    private String usuarioId;

    public void agregarItem(ItemCarrito item) {
        items.add(item);
    }

    public void setUsuarioId(String usuarioId) {
        this.usuarioId = usuarioId;
    }

    public int obtenerTotal() {
        return items.stream()
            .mapToInt(item -> item.getPrecio() * item.getCantidad())
            .sum();
    }
}
```

```java
@Service
public class ServicioTienda {

    @Autowired
    private ApplicationContext context;

    public CarritoCompra crearCarrito(String usuarioId) {
        // Cada llamada crea una nueva instancia
        CarritoCompra carrito = context.getBean(CarritoCompra.class);
        carrito.setUsuarioId(usuarioId);
        return carrito;
    }
}
```

Los beans prototype son apropiados para objetos con estado que representan datos de una sesion o transaccion especifica: carritos de compra, formularios de edicion, objetos de trabajo temporales. Sin embargo, inyectar un bean prototype en un bean singleton requiere atencion especial, ya que la referencia se resuelve una sola vez y se reutiliza para todas las llamadas.

> Cuando se inyecta un bean prototype en un singleton, se crea una unica instancia del prototype durante la inicializacion del singleton. Si se necesita una nueva instancia del prototype en cada invocacion, se debe usar `ObjectFactory<T>` o `Provider<T>`:

```java
@Service
public class ServicioProceso {

    private final ObjectFactory<CarritoCompra> carritoFactory;

    public ServicioProceso(ObjectFactory<CarritoCompra> carritoFactory) {
        this.carritoFactory = carritoFactory;
    }

    public void procesar Pedido(String usuarioId) {
        // Cada llamada crea una nueva instancia de CarritoCompra
        CarritoCompra carrito = carritoFactory.getObject();
        carrito.setUsuarioId(usuarioId);
        // ...
    }
}
```

## Request: una instancia por request HTTP

El scope `request` crea una nueva instancia del bean para cada request HTTP. La instancia vive durante el ciclo de vida de la request y se destruye automaticamente cuando la response se envia al cliente. Este scope solo esta disponible en un contexto web (`WebApplicationContext`).

```java
@Component
@RequestScope
public class ContextoSolicitud {

    private final Map<String, Object> atributos = new HashMap<>();
    private LocalDateTime momentoRecepcion;

    @PostConstruct
    public void inicializar() {
        this.momentoRecepcion = LocalDateTime.now();
    }

    public void setAtributo(String clave, Object valor) {
        atributos.put(clave, valor);
    }

    public Object getAtributo(String clave) {
        return atributos.get(clave);
    }
}
```

`@RequestScope` es un atajo para `@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)`. El `proxyMode` es necesario porque el bean request es mas corto que el bean singleton que lo consume. Spring crea un proxy que delega al bean real de la request actual.

```java
@RestController
public class PedidoController {

    private final ContextoSolicitud contexto;

    public PedidoController(ContextoSolicitud contexto) {
        this.contexto = contexto;
    }

    @GetMapping("/pedido/{id}")
    public PedidoDTO obtenerPedido(@PathVariable Long id) {
        contexto.setAtributo("pedidoId", id);
        // contexto se resuelve al bean de la request actual
        return servicioPedido.obtener(id);
    }
}
```

## Session: una instancia por sesion HTTP

El scope `session` crea una nueva instancia del bean para cada sesion HTTP del usuario. La instancia persiste mientras la sesion este activa y se destruye cuando la sesion expira o se invalida explicitamente.

```java
@Component
@SessionScope
public class SesionUsuario {

    private UsuarioDTO usuario;
    private List<String> permisos;

    public void iniciarSesion(UsuarioDTO usuario) {
        this.usuario = usuario;
        this.permisos = usuario.getPermisos();
    }

    public boolean tienePermiso(String permiso) {
        return permisos != null && permisos.contains(permiso);
    }

    public void cerrarSesion() {
        this.usuario = null;
        this.permisos = null;
    }
}
```

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final SesionUsuario sesion;

    public AuthController(SesionUsuario sesion) {
        this.sesion = sesion;
    }

    @PostMapping("/login")
    public ResponseEntity<Void> login(@RequestBody LoginRequest request) {
        UsuarioDTO usuario = authService.autenticar(request);
        sesion.iniciarSesion(usuario);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/perfil")
    public ResponseEntity<UsuarioDTO> perfil() {
        return ResponseEntity.ok(sesion.getUsuario());
    }
}
```

> Los beans con scope de sesion pueden causar fugas de memoria si la aplicacion almacena objetos grandes en la sesion y las sesiones no expiran adecuadamente. En aplicaciones con multiples instancias (clusters), las sesiones deben ser distribuidas (via Spring Session con Redis o JDBC) para garantizar la consistencia.

## Application: una instancia por ServletContext

El scope `application` crea una instancia del bean por `ServletContext` de la aplicacion web. A diferencia del scope singleton, que es unico por `ApplicationContext`, el scope `application` es unico por contexto servlet. En una aplicacion con multiples contextos de Spring (por ejemplo, un contexto raiz y uno servlet), el scope `application` garantiza una sola instancia compartida entre todos ellos.

```java
@Component
@ApplicationScope
public class EstadisticasAplicacion {

    private final AtomicLong totalPeticiones = new AtomicLong(0);
    private final AtomicLong totalErrores = new AtomicLong(0);

    public void registrarPeticion() {
        totalPeticiones.incrementAndGet();
    }

    public void registrarError() {
        totalErrores.incrementAndGet();
    }

    public long getTotalPeticiones() {
        return totalPeticiones.get();
    }
}
```

## Websocket: una instancia por WebSocket

El scope `websocket` crea una instancia del bean por cada conexion WebSocket. La instancia vive mientras la conexion este activa y se destruye cuando se cierra. Este scope es util para mantener estado especifico de cada cliente conectado.

```java
@Component
@Scope(scopeName = "websocket", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class SesionChat {

    private String usuario;
    private final List<MensajeChat> mensajes = new ArrayList<>();

    public void setUsuario(String usuario) {
        this.usuario = usuario;
    }

    public void agregarMensaje(MensajeChat mensaje) {
        mensajes.add(mensaje);
    }
}
```

## @Scope y scopes personalizados

La anotacion `@Scope` permite especificar el scope de un bean definido via `@Bean` o `@Component`. Ademas de los scopes incorporados, Spring permite crear scopes personalizados implementando la interface `Scope` y registrandolos en el contenedor.

```java
@Configuration
public class ConfiguracionScopes {

    @Bean
    @Scope("prototype")
    public Procesador procesador() {
        return new Procesador();
    }

    @Bean
    @Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
    public AnalisisRequest analisisRequest() {
        return new AnalisisRequest();
    }
}
```

```java
// Registro de un scope personalizado
@Component
public class OrganizacionScope implements Scope {

    private final ThreadLocal<Map<String, Object>> scopedObjects = new ThreadLocal<>();

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> scope = scopedObjects.get();
        if (scope == null) {
            scope = new HashMap<>();
            scopedObjects.set(scope);
        }
        return scope.computeIfAbsent(name, k -> objectFactory.getObject());
    }

    @Override
    public Object remove(String name) {
        Map<String, Object> scope = scopedObjects.get();
        return scope != null ? scope.remove(name) : null;
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        // Registrar callback para limpieza
    }

    @Override
    public Object resolveContextualObject(String key) {
        return null;
    }

    @Override
    public String getConversationId() {
        return Thread.currentThread().getName();
    }
}
```

```java
// Registrar el scope en el contenedor
@Configuration
public class ConfiguracionCustomScope {

    @Bean
    public static CustomScopeConfigurer customScopeConfigurer() {
        CustomScopeConfigurer configurer = new CustomScopeConfigurer();
        configurer.addScope("organizacion", new OrganizacionScope());
        return configurer;
    }
}
```

> Los scopes personalizados son avanzados y rara vez necesarios en aplicaciones convencionales. Antes de implementar un scope personalizado, evaluar si un scope existente (request, session, o prototype con `ObjectFactory`) puede resolver el caso de uso. Los scopes personalizados requieren gestion manual de la destruccion de objetos y pueden introducir fugas de memoria si no se implementan correctamente.

