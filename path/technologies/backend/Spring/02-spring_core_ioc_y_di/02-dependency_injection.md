# Dependency Injection (DI)

La Inyeccion de Dependencias es el mecanismo mediante el cual el contenedor IoC de Spring resuelve y asigna las dependencias de un bean. Mientras que IoC es el patron de diseño general (delegar el control de creacion y configuracion de objetos a un framework), DI es la implementacion concreta de ese patron en Spring: el contenedor "inyecta" las referencias a otros beans en el bean que las necesita. Esta distincion es importante porque IoC es un concepto amplio que puede implementarse de multiples formas (service locator, template method, etc.), pero DI es la estrategia especifica que Spring adopta como mecanismo principal.

El beneficio fundamental de DI es la desacoplacion. Un bean no necesita conocer la implementacion concreta de sus dependencias ni como se crean. Solo necesita declarar que necesita una instancia de un tipo determinado, y el contenedor se encarga de proporcionarla. Esto facilita las pruebas (se pueden inyectar mocks), la mantenibilidad (se pueden cambiar implementaciones sin modificar el bean que las consume) y la composicion (se pueden crear configuraciones diferentes ensamblando distintas implementaciones).

## Constructor Injection: la forma recomendada

La inyeccion por constructor consiste en declarar las dependencias como parametros del constructor de la clase. Spring detecta el constructor, resuelve los beans que coinciden con los tipos de los parametros y los proporciona al instanciar la clase. Esta es la forma de inyeccion recomendada por el equipo de Spring y por la mayoria de las guias de estilo de la comunidad.

```java
@Service
public class ServicioPedido {

    private final RepositorioPedido repositorio;
    private final ServicioInventario inventario;
    private final Notificador notificador;

    // Spring detecta este constructor automaticamente
    // porque es el unico constructor de la clase
    public ServicioPedido(RepositorioPedido repositorio,
                          ServicioInventario inventario,
                          Notificador notificador) {
        this.repositorio = repositorio;
        this.inventario = inventario;
        this.notificador = notificador;
    }

    public Pedido crearPedido(CrearPedidoRequest request) {
        inventario.verificarDisponibilidad(request.getItems());
        Pedido pedido = new Pedido(request);
        repositorio.guardar(pedido);
        notificador.enviarConfirmacion(pedido);
        return pedido;
    }
}
```

Cuando una clase tiene un unico constructor, Spring lo inyecta automaticamente sin necesidad de anotacion `@Autowired`. El contenedor analiza los tipos de los parametros del constructor, resuelve los beans correspondientes y los proporciona. Si no encuentra un bean que coincida con alguno de los tipos, lanza una excepcion `NoSuchBeanDefinitionException` durante la inicializacion.

La anotacion `@Autowired` en el constructor es opcional cuando existe un unico constructor:

```java
// Estas dos formas son equivalentes

// Sin @Autowired (Spring lo detecta automaticamente)
public ServicioPedido(RepositorioPedido repositorio) {
    this.repositorio = repositorio;
}

// Con @Autowired (explicito pero redundante)
@Autowired
public ServicioPedido(RepositorioPedido repositorio) {
    this.repositorio = repositorio;
}
```

Las ventajas de la inyeccion por constructor son multiples. En primer lugar, las dependencias pueden declararse como `final`, lo que garantiza la inmutabilidad del bean una vez creado. En segundo lugar, las dependencias nunca pueden ser nulas (a menos que se use `@Nullable`), lo que elimina un error comun de ejecucion. En tercer lugar, las dependencias se resuelven en el momento de la instanciacion, no en un momento posterior, lo que facilita la validacion de la configuracion. Finalmente, los tests unitarios son directos: se puede instanciar la clase pasando mocks como argumentos al constructor, sin necesidad de un contenedor de Spring.

```java
// Test unitario sin contenedor de Spring
@Test
void crearPedido_cuandoStockSuficiente_guardaPedido() {
    RepositorioPedido mockRepositorio = mock(RepositorioPedido.class);
    ServicioInventario mockInventario = mock(ServicioInventario.class);
    Notificador mockNotificador = mock(Notificador.class);

    ServicioPedido servicio = new ServicioPedido(mockRepositorio, mockInventario, mockNotificador);

    CrearPedidoRequest request = new CrearPedidoRequest(List.of(new ItemPedido("SKU-1", 2)));
    servicio.crearPedido(request);

    verify(mockRepositorio).guardar(any(Pedido.class));
}
```

## Setter Injection: dependencias opcionales

La inyeccion por setter consiste en proporcionar metodos setter que el contenedor invoca despues de instanciar la clase. Esta forma es adecuada para dependencias que pueden ser opcionales o que pueden cambiar despues de la creacion del bean.

```java
@Service
public class ServicioReporte {

    private final RepositorioReporte repositorio;
    private ConfiguradorReporte configurador; // Dependencia opcional

    public ServicioReporte(RepositorioReporte repositorio) {
        this.repositorio = repositorio; // Obligatoria via constructor
    }

    @Autowired
    public void setConfigurador(ConfiguradorReporte configurador) {
        this.configurador = configurador;
    }

    public byte[] generarReporte(Long id) {
        Reporte reporte = repositorio.obtenerPorId(id);
        if (configurador != null) {
            configurador.aplicarFormato(reporte);
        }
        return reporte.exportar();
    }
}
```

> La inyeccion por setter tiene una desventaja significativa: las dependencias no pueden ser `final` y pueden ser nulas. Esto debilita las garantias del compilador y requiere verificaciones null en tiempo de ejecucion. El equipo de Spring desaconseja esta forma para dependencias obligatorias.

## Field Injection: no recomendada

La inyeccion por campo consiste en anotar directamente un campo con `@Autowired` para que Spring asigne el valor. Esta es la forma mas directa pero tambien la mas problematica.

```java
@Service
public class ServicioNotificacion {

    @Autowired
    private RepositorioNotificacion repositorio; // No recomendado

    @Autowired
    private PlantillaFactory plantillaFactory; // No recomendado

    public void notificar(Usuario usuario, String mensaje) {
        Plantilla plantilla = plantillaFactory.crear("notificacion");
        repositorio.guardar(new Notificacion(usuario, plantilla.render(mensaje)));
    }
}
```

Los problemas de la inyeccion por campo son multiples. En primer lugar, oculta las dependencias de la clase: un desarrollador que observe la firma del constructor no puede saber que dependencias tiene la clase. En segundo lugar, impide la creacion de objetos fuera del contenedor de Spring, lo que dificulta enormemente las pruebas unitarias. Para testear una clase con inyeccion por campo, se necesita o bien usar reflection para asignar el campo (lo cual es fragil y viola el encapsulamiento) o usar un contexto de Spring completo en pruebas que deberian ser unitarias. En tercer lugar, crea un acoplamiento estrecho entre la clase y el contenedor de Spring, ya que la clase depende de la anotacion `@Autowired` para funcionar.

> El propio Rod Johnson, creador de Spring, declaro en 2019 que la inyeccion por campo habia sido un error de disenos. Si bien el framework continua soportandola por razones de compatibilidad, la recomendacion oficial es utilizar inyeccion por constructor.

## @Qualifier y @Primary: resolucion de multiples beans

Cuando existen multiples implementaciones de una misma interfaz, el contenedor necesita criterios adicionales para determinar cual inyectar. La anotacion `@Qualifier` permite especificar el nombre del bean a inyectar.

```java
public interface EstrategiaCache {
    void put(String clave, Object valor);
    Object get(String clave);
}

@Component("cacheRedis")
public class CacheRedis implements EstrategiaCache {
    // Implementacion con Redis
}

@Component("cacheMemcached")
public class CacheMemcached implements EstrategiaCache {
    // Implementacion con Memcached
}

@Service
public class ServicioCatalogo {

    private final EstrategiaCache cache;

    public ServicioCatalogo(@Qualifier("cacheRedis") EstrategiaCache cache) {
        this.cache = cache;
    }
}
```

`@Primary` indica el bean que debe inyectarse por defecto cuando existe mas de uno del mismo tipo, sin necesidad de especificar `@Qualifier`. Es util cuando una implementacion es claramente la preferida y la otra es una alternativa excepcional.

```java
@Component
@Primary
public class CacheRedis implements EstrategiaCache {
    // Esta sera inyectada por defecto
}

@Component
public class CacheLocal implements EstrategiaCache {
    // Solo se inyecta cuando se especifica @Qualifier
}
```

## @Autowired vs @Resource vs @Inject

Spring soporta multiples anotaciones para la inyeccion de dependencias. `@Autowired` es una anotacion propiamente de Spring que soporta inyeccion por constructor, setter y campo, y permite marcar dependencias como opcionales con `@Nullable`. `@Resource` es una anotacion de Jakarta (JSR-250) que inyecta por nombre por defecto (busca un bean con el nombre que coincida con el nombre del campo). `@Inject` es una anotacion de Jakarta (JSR-330) equivalente a `@Autowired` pero sin soporte para `required = false`.

```java
@Service
public class EjemploAnotaciones {

    // Spring: inyeccion por tipo, fallback por nombre
    @Autowired
    private ServicioA servicioA;

    // Jakarta: inyeccion por nombre primero, luego por tipo
    @Resource
    private ServicioB servicioB;

    // Jakarta: inyeccion por tipo, sin soporte para opcional
    @Inject
    private ServicioC servicioC;
}
```

La recomendacion practica es utilizar `@Autowired` (o preferiblemente, inyeccion por constructor sin anotacion) de forma consistente. Mezclar anotaciones de diferentes especificaciones en el mismo proyecto genera confusion y dificulta la portabilidad.

## @Lazy: inicializacion perezosa

La anotacion `@Lazy` indica que el contenedor no debe instanciar el bean hasta que se solicite por primera vez. Esto es util para beans cuya inicializacion es costosa o que dependen de recursos que no estan disponibles durante el arranque de la aplicacion.

```java
@Service
public class ServicioReporte {

    private final RepositorioReporte repositorio;

    public ServicioReporte(@Lazy RepositorioReporte repositorio) {
        this.repositorio = repositorio;
        // repositorio es un proxy que se resuelve
        // solo cuando se invoca un metodo
    }
}
```

Cuando se marca un bean como `@Lazy`, Spring crea un proxy del bean. Este proxy delega las llamadas al bean real solo cuando se invoca por primera vez. Esto retrasa la creacion del bean y todas sus dependencias hasta el momento de uso, reduciendo el tiempo de arranque de la aplicacion.

```java
@Configuration
public class Configuracion {

    @Bean
    @Lazy
    public ServicioExterno servicioExterno() {
        // Esta inicializacion es costosa (conexion a API externa)
        return new ServicioExterno(configurarConexion());
    }

    private ConfiguracionConexion configurarConexion() {
        // Logica compleja de configuracion
        return new ConfiguracionConexion();
    }
}
```

> `@Lazy` en el nivel de bean es diferente a `@Lazy` a nivel de `@Configuration`. En un `@Configuration` con `@Lazy`, todos los beans definidos en esa clase se vuelven lazy. En un bean individual, solo ese bean concreto es lazy. El uso excesivo de `@Lazy` puede enmascarar problemas de configuracion que solo se manifiestan en produccion.

