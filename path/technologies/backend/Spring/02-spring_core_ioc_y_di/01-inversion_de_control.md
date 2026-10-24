# Inversion de Control (IoC)

La Inversion de Control es el patron de diseño fundamental sobre el que se construye Spring Framework. En un programa convencional, el flujo de control esta determinado por el propio codigo: una clase instancia sus dependencias, configura sus colaboradores y gestiona su propio ciclo de vida. En un contenedor IoC, esta responsabilidad se delega a un framework externo que se encarga de crear los objetos, resolver sus dependencias y gestionar su ciclo de vida completo. La "inversion" se refiere precisamente a esta transferencia de responsabilidad: el codigo deja de controlar la creacion de sus dependencias y pasa a declararlas.

Antes de existir un contenedor IoC, un servicio de negocio que necesitaba acceder a una base de datos tenia que crear internamente la conexion, instanciar el DAO correspondiente y manejar la transaccion. Esto generaba un acoplamiento estrecho entre capas: el servicio conocia la implementacion concreta del DAO, los parametros de conexion a la base de datos y la logica de transacciones. El contenedor IoC elimina este acoplamiento al asumir la responsabilidad de ensamblar estos objetos y resolver las dependencias entre ellos.

## El contenedor de IoC en Spring

El contenedor de IoC de Spring es la pieza central del framework. Su responsabilidad es leer metadatos de configuracion (ya sea anotaciones Java, XML o codigo de configuracion programatica), instanciar los beans segun esas definiciones, inyectar las dependencias declaradas, invocar los callbacks de ciclo de vida, y entregar los beans listos para su uso cuando se soliciten.

El proceso comienza con la definicion de los metadatos. Estos metadatos describen que clases deben ser gestionadas por el contenedor, como deben ser configuradas y que dependencias deben ser inyectadas. Una vez que el contenedor es inicializado con estos metadatos, entra en un ciclo de vida activo donde puede crear, configurar, entregar y destruir beans segun sea necesario.

```java
// Definicion de metadatos via anotaciones
@Configuration
public class AppConfig {

    @Bean
    public RepositorioUsuario repositorioUsuario() {
        return new RepositorioUsuarioJdbc();
    }

    @Bean
    public ServicioUsuario servicioUsuario(RepositorioUsuario repositorio) {
        return new ServicioUsuarioImpl(repositorio);
    }
}

// Uso del contenedor
public class Aplicacion {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        ServicioUsuario servicio = context.getBean(ServicioUsuario.class);
        servicio.registrar(new Usuario("Juan", "juan@email.com"));
    }
}
```

## BeanFactory: el contenedor basico

`BeanFactory` es la interfaz mas elemental del contenedor IoC de Spring. Define las operaciones basicas para acceder a los beans: `getBean()`, `containsBean()`, `isSingleton()`, `isPrototype()` y `getType()`. La implementacion por defecto es `DefaultListableBeanFactory`, que soporta todas las funcionalidades del contenedor pero con una caracteristica diferenciadora: lazy loading.

En un `BeanFactory`, los beans no se instancian hasta que se solicitan por primera vez via `getBean()`. Esto es util en escenarios donde el consumo de memoria es critico o donde el contenedor debe arrancar rapidamente sin instanciar todos los beans. Sin embargo, implica que errores de configuracion (como dependencias circulares o beans mal definidos) no se detectan hasta que se accede al bean problematico.

```java
// BeanFactory con lazy loading
BeanFactory factory = new DefaultListableBeanFactory();
// Los beans NO se han instanciado aun

XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions("beans.xml");

// El bean se instancia solo ahora, al ser solicitado
MiBean bean = factory.getBean(MiBean.class);
```

## ApplicationContext: el contenedor completo

`ApplicationContext` extiende `BeanFactory` con una serie de capacidades que lo convierten en la interfaz preferida para el uso en aplicaciones reales. Estas capacidades incluyen:

- **Eager loading**: los beans singleton se instancian durante la inicializacion del contenedor, lo que permite detectar errores de configuracion tempranamente.
- **Resolucion de internacionalizacion**: implementa `MessageSource` para resolver mensajes de internacionalizacion (i18n).
- **Publicacion de eventos**: implementa `ApplicationEventPublisher` para permitir la comunicacion entre beans mediante un patron observer.
- **Carga de recursos externos**: implementa `ResourceLoader` para acceder a archivos, URLs y其它 recursos de forma abstracta.
- **Entorno configurable**: implementa `Environment` para acceder a propiedades del sistema y profiles activos.

```java
// ApplicationContext con eager loading
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// TODOS los beans singleton ya se han instanciado
// Si hay un error de configuracion, falla aqui

// Resolucion de internacionalizacion
String mensaje = context.getMessage("bienvenida", new Object[]{"Juan"}, Locale.forLanguageTag("es"));

// Publicacion de eventos
context.publishEvent(new UsuarioRegistradoEvent(usuario));

// Acceso a propiedades
String url = context.getEnvironment().getProperty("database.url");
```

### Tipos de ApplicationContext

Spring ofrece multiples implementaciones de `ApplicationContext` adaptadas a diferentes entornos de ejecucion:

```java
// Clasepath: lee configuracion desde el classpath
ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");

// FileSystem: lee configuracion desde una ruta del sistema de archivos
ApplicationContext context = new FileSystemXmlApplicationContext("/opt/config/beans.xml");

// Anotaciones Java: usa una clase @Configuration como fuente
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

// Web: contexto para aplicaciones web
GenericWebApplicationContext context = new GenericWebApplicationContext();
context.registerBean(Configuracion.class);
context.refresh();
```

`ClassPathXmlApplicationContext` busca los archivos de configuracion en el classpath de la aplicacion, lo que lo hace portable entre entornos. `FileSystemXmlApplicationContext` resuelve las rutas相对于el sistema de archivos del servidor, lo que lo hace util para configuraciones externas a la aplicacion. `AnnotationConfigApplicationContext` es el mas utilizado en la practica moderna, ya que elimina la necesidad de archivos XML. `GenericWebApplicationContext` se usa internamente por Spring MVC y Spring Boot para el ciclo de vida de una aplicacion web.

> `AnnotationConfigApplicationContext` soporta multiples clases de configuracion como entrada, permitiendo dividir la configuracion en modulos separados. Cada clase anotada con `@Configuration` puede ser registrada, y el contenedor ensamblara los beans de todas ellas.

## BeanFactory vs ApplicationContext

La diferencia practica entre `BeanFactory` y `ApplicationContext` se resume en tres areas clave:

**Inicializacion**: `BeanFactory` es lazy por defecto. Los beans se crean bajo demanda. `ApplicationContext` es eager para los beans singleton. Esto significa que un `ApplicationContext` tarda mas en iniciarse pero detecta problemas de configuracion inmediatamente.

**Funcionalidad**: `ApplicationContext` anade capacidades que `BeanFactory` no tiene: i18n, eventos, carga de recursos y entorno configurable. En una aplicacion real, estas capacidades son necesarias con frecuencia suficiente como para justificar el uso de `ApplicationContext`.

**Rendimiento**: `BeanFactory` tiene un consumo de memoria menor porque no instancian beans innecesariamente. Sin embargo, la diferencia es marginal en la mayoria de las aplicaciones y solo se justifica en entornos con restricciones extremas de memoria.

```java
// Diferencia en comportamiento

// BeanFactory: lazy - el bean no existe hasta que se pide
BeanFactory factory = new DefaultListableBeanFactory();
// No falla aqui aunque el XML tenga errores
MiBean bean = factory.getBean(MiBean.class); // Falla aqui si hay error

// ApplicationContext: eager - todos los beans se crean al iniciar
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// Falla aqui si hay error en la configuracion
```

En la practica moderna, `ApplicationContext` es la unica interfaz que se utiliza. Incluso Spring Boot, que oculta mucho del boilerplate de configuracion, utiliza internamente un `AnnotationConfigApplicationContext` (o una variante web) como contenedor de la aplicacion.

> Si bien `BeanFactory` sigue existiendo por razones de compatibilidad, los libros y documentacion oficial recomiendan `ApplicationContext` para todos los casos de uso. La unica excepcion seria un escenario donde se necesita una carga extremadamente diferida de beans en un entorno con restricciones de memoria severas.

