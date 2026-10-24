# Ciclo de vida del bean

El ciclo de vida de un bean en Spring es el conjunto de fases por las que atraviesa desde su creacion hasta su destruccion. Comprender este ciclo es esencial para implementar correctamente la inicializacion de recursos, la limpieza de conexiones y la integracion con frameworks de terceros. Cada fase del ciclo ofrece puntos de extension donde el desarrollador puede inyectar logica personalizada, bien sea a traves de interfaces del framework, anotaciones estandar o metodos personalizados.

El contenedor IoC de Spring no simplemente crea un objeto y lo devuelve. Existe un proceso complejo de instanciacion, configuracion, inicializacion y gestion que involucra multiples callbacks, orden de precedencia definido y puntos de intercepcion que permiten a Aspectos y BeanPostProcessors modificar el comportamiento del bean en cualquier momento.

## Diagrama del ciclo de vida

El ciclo de vida completo de un bean singleton en Spring sigue las siguientes fases, en estricto orden:

```
1. Instanciacion (constructor)
2. Inyeccion de propiedades (DI)
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory()
5. ApplicationContextAware.setApplicationContext()
6. BeanPostProcessor.postProcessBeforeInitialization()
7. @PostConstruct
8. InitializingBean.afterPropertiesSet()
9. Metodo init personalizado
10. BeanPostProcessor.postProcessAfterInitialization()
11. Uso del bean (servicio activo)
12. @PreDestroy
13. DisposableBean.destroy()
14. Metodo destroy personalizado
```

## Fase 1: Instanciacion

El contenedor crea una instancia del bean invocando su constructor. Si el bean tiene multiples constructores, el contenedor selecciona uno basado en las anotaciones `@Autowired` o la resolucion por tipo. Si no hay ambiguedad y existe un unico constructor, se utiliza ese. Si hay ambiguedad sin resolver, el contenedor lanza una excepcion `BeanCreationException`.

```java
public class ServicioPago {

    // Spring invoca este constructor
    public ServicioPago(GatewayPago gateway) {
        this.gateway = gateway;
        System.out.println("1. Constructor invocado");
    }
}
```

Durante la instanciacion, el objeto aun no tiene sus dependencias inyectadas. Los campos declarados como `final` y asignados en el constructor estan disponibles, pero los campos inyectados via setter o campo aun son nulos.

## Fase 2: Inyeccion de propiedades

Despues de instanciar el bean, el contenedor resuelve y asigna todas las dependencias declaradas. Este proceso incluye la inyeccion por constructor (si no se hizo en la fase anterior), por setter y por campo. Tambien se procesan las anotaciones `@Value` para inyectar propiedades del entorno o expresiones SpEL.

```java
@Service
public class ServicioEnvio {

    private final RepositorioEnvio repositorio; // Inyectado via constructor

    @Value("${app.envio.tiempo-maximo:24h}")
    private String tiempoMaximo; // Inyectado via @Value

    @Autowired
    private ConfiguradorEnvio configurador; // Inyectado via campo

    public ServicioEnvio(RepositorioEnvio repositorio) {
        this.repositorio = repositorio;
    }
}
```

> En esta fase, el bean esta en un estado intermedio: ya existe como objeto en memoria pero podria no estar completamente inicializado. Los metodos de negocio no deberian invocarse hasta que el ciclo de vida complete las fases de inicializacion.

## Fases 3-5: Aware interfaces

Spring proporciona un conjunto de interfaces "Aware" que permiten al bean acceder a recursos del contenedor. Estas interfaces son invocadas en un orden especifico despues de la inyeccion de propiedades:

- `BeanNameAware.setBeanName(String name)`: proporciona el nombre del bean dentro del contenedor.
- `BeanFactoryAware.setBeanFactory(BeanFactory factory)`: proporciona referencia al `BeanFactory` que contiene el bean.
- `ApplicationContextAware.setApplicationContext(ApplicationContext ctx)`: proporciona referencia al `ApplicationContext`.
- `EnvironmentAware.setEnvironment(Environment env)`: proporciona acceso al `Environment`.
- `ResourceLoaderAware.setResourceLoader(ResourceLoader loader)`: proporciona acceso al cargador de recursos.
- `ApplicationEventPublisherAware.setApplicationEventPublisher(...)`: permite publicar eventos.
- `MessageSourceAware.setMessageSource(MessageSource ms)`: permite resolver mensajes i18n.

```java
@Component
public class ServicioConContexto implements ApplicationContextAware, BeanNameAware {

    private ApplicationContext context;
    private String nombreBean;

    @Override
    public void setBeanName(String name) {
        this.nombreBean = name;
        System.out.println("3. BeanNameAware - nombre: " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
        System.out.println("5. ApplicationContextAware - contexto recibido");
    }

    public void usarContexto() {
        // Acceder a otros beans desde el contexto
        ServicioOtro otro = context.getBean(ServicioOtro.class);
    }
}
```

> Las interfaces Aware son un patron de Spring que permite un acoplamiento optativo con el framework. En la mayoria de los casos, es preferible inyectar directamente los recursos que se necesitan (ApplicationContext, BeanFactory, etc.) en lugar de implementar estas interfaces.

## Fase 6: BeanPostProcessor - antes de inicializacion

`BeanPostProcessor` es uno de los mecanismos de extension mas poderosos de Spring. Permite interceptar y modificar los beans despues de su creacion pero antes de su inicializacion. El metodo `postProcessBeforeInitialization` se invoca para cada bean que pasa por el contenedor.

```java
@Component
public class MiBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof ServicioConMetricas) {
            System.out.println("6. PostProcessor BEFORE - procesando: " + beanName);
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof ServicioConMetricas) {
            System.out.println("10. PostProcessor AFTER - procesando: " + beanName);
        }
        return bean;
    }
}
```

Los BeanPostProcessors se ejecutan para todos los beans del contenedor (a menos que se filtre explicitamente). Son la base para muchas funcionalidades de Spring: la resolucion de `@Autowired`, la creacion de proxies AOP, la activacion de `@Scheduled` y `@Async`, y la deteccion de anotaciones personalizadas.

## Fases 7-9: Inicializacion

Estas tres fases representan los puntos de inicializacion disponibles para el desarrollador. Se ejecutan en este orden estricto:

**@PostConstruct (JSR-250)**: anotacion estandar de Jakarta que indica un metodo que debe ejecutarse despues de la inyeccion de dependencias. Es la forma mas comun de inicializacion en Spring moderno.

**InitializingBean.afterPropertiesSet()**: interface de Spring que permite ejecutar logica despues de que todas las propiedades hayan sido inyectadas. Es una alternativa a `@PostConstruct` pero acopla la clase al framework.

**Metodo personalizado**: un metodo arbitrario especificado via `@Bean(initMethod = "miMetodo")` o via XML `init-method="miMetodo"`. No acopla la clase a ninguna interface o anotacion de Spring.

```java
@Service
public class ConexionBaseDatos {

    private Connection connection;

    // Opcion 1: @PostConstruct (recomendado)
    @PostConstruct
    public void inicializar() {
        this.connection = DriverManager.getConnection(url, user, pass);
        System.out.println("7. @PostConstruct - conexion establecida");
    }
}

// Opcion 2: InitializingBean
@Component
public class CacheInicializador implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
        // Cargar datos iniciales en cache
        System.out.println("8. InitializingBean.afterPropertiesSet()");
    }
}

// Opcion 3: metodo personalizado
@Configuration
public class Configuracion {

    @Bean(initMethod = "inicializarCache")
    public CacheManager cacheManager() {
        return new CacheManager();
    }
}

public class CacheManager {
    public void inicializarCache() {
        System.out.println("9. init-method personalizado");
    }
}
```

> `@PostConstruct` es la forma preferida de inicializacion porque es estandar (no acopla al framework), es declarativa y no requiere configuracion adicional. Sin embargo, cuando se necesita acceder al `ApplicationContext` o al `BeanFactory`, las interfaces Aware se invocan antes que `@PostConstruct`.

## Fase 10: BeanPostProcessor - despues de inicializacion

Despues de ejecutar los metodos de inicializacion, `BeanPostProcessor.postProcessAfterInitialization` se invoca nuevamente para cada bean. Este es el punto donde los BeanPostProcessors crean proxies AOP para los beans que los necesiten. Si un bean tiene metodos anotados con `@Transactional`, `@Cacheable` o `@Async`, en esta fase se reemplaza la referencia del bean por un proxy que intercepta las llamadas a esos metodos.

## Fase 11: Uso del bean

El bean esta completamente inicializado y listo para ser utilizado. Este es el estado en el que el bean permanece la mayoria de su vida util, respondiendo a llamadas de otros beans y procesando solicitudes.

## Fases 12-14: Destruccion

Cuando el contenedor se apaga (por ejemplo, al cerrar la aplicacion), los beans singleton pasan por su fase de destruccion. Este proceso incluye tres puntos de extension:

**@PreDestroy (JSR-250)**: anotacion estandar que indica un metodo a ejecutar antes de la destruccion del bean. Comunmente se usa para cerrar conexiones, liberar recursos y limpiar caches.

**DisposableBean.destroy()**: interface de Spring equivalente a `@PreDestroy` pero acoplada al framework.

**Metodo personalizado de destruccion**: especificado via `@Bean(destroyMethod = "cerrar")` o `destroy-method` en XML.

```java
@Service
public class ConexionPool {

    private DataSource dataSource;

    @PostConstruct
    public void inicializar() {
        this.dataSource = crearDataSource();
    }

    @PreDestroy
    public void limpiar() {
        if (dataSource != null) {
            ((Closeable) dataSource).close();
            System.out.println("12. @PreDestroy - pool cerrado");
        }
    }
}
```

> Spring detecta automaticamente metodos con la firma `close()` o `shutdown()` y los invoca durante la destruccion sin necesidad de anotacion `@PreDestroy`. Si el metodo de limpieza no debe ejecutarse, se puede desactivar con `@Bean(destroyMethod = "")`.

