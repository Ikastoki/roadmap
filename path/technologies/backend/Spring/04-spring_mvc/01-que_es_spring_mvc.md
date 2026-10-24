# Que es Spring MVC

Spring MVC es el módulo de Spring Framework que implementa el patrón Model-View-Controller (MVC) para aplicaciones web. Proporciona una arquitectura completa y bien diseñada para desarrollar aplicaciones web flexibles, donde la lógica de presentación está claramente separada de la lógica de negocio y el modelo de datos. Spring MVC es la base sobre la que se construyen aplicaciones web modernas con Spring.

El patrón Front Controller es fundamental en la arquitectura de Spring MVC. En este patrón, un único servlet maneja todas las peticiones HTTP y las distribuye a los handlers apropiados. En Spring MVC, este rol lo desempeña `DispatcherServlet`, que actúa como punto de entrada central para todas las peticiones web. El `DispatcherServlet` coordina todo el proceso de manejo de peticiones, desde la recepción hasta la generación de la respuesta.

`DispatcherServlet` es un servlet que extiende `HttpServlet` y está configurado para interceptar peticiones que coincidan con un patrón de URL específico. En una aplicación típica, se configura para manejar todas las peticiones (`/`), pero también se pueden configurar múltiples `DispatcherServlet` con diferentes patrones para separar funcionalidades. El `DispatcherServlet` utiliza varios componentes auxiliares para procesar cada petición de manera modular.

El flujo de procesamiento de peticiones en Spring MVC sigue un camino bien definido. Cuando una petición HTTP llega al servidor, el `DispatcherServlet` la recibe y consulta al `HandlerMapping` para determinar qué controller debe manejarla. El `HandlerMapping` analiza la URL, los headers y otros atributos de la petición para encontrar el handler adecuado. Una vez identificado el handler, el `DispatcherServlet` delega la ejecución al `HandlerAdapter`.

El `HandlerAdapter` es responsable de invocar al controller con los argumentos correctos. Este componente maneja la conversión de parámetros de la petición a los tipos esperados por el controller, la validación de datos y la llamada al método del controller. El `HandlerAdapter` abstrae la complejidad de invocar diferentes tipos de handlers, proporcionando una interfaz uniforme para el `DispatcherServlet`.

Después de que el controller ejecuta la lógica de negocio y retorna un modelo de datos, el `DispatcherServlet` utiliza el `ViewResolver` para resolver el nombre de la vista a un objeto `View` concreto. El `ViewResolver` mapea nombres de vista a plantillas específicas, como JSP, Thymeleaf o FreeMarker. Este mecanismo permite cambiar el motor de plantillas sin modificar el código del controller.

La `View` es responsable de renderizar el modelo de datos en el formato de respuesta apropiado. Para aplicaciones web tradicionales, esto significa generar HTML. Para aplicaciones REST, el `ViewResolver` puede retornar un `MappingJackson2JsonView` que serializa el modelo a JSON. En aplicaciones REST modernas, el retorno directo del objeto con `@ResponseBody` o el uso de `ResponseEntity` es más común que el mecanismo de vistas.

```java
@Controller
@RequestMapping("/usuarios")
public class UsuarioController {
    
    private final UsuarioService usuarioService;
    
    @GetMapping
    @ResponseBody
    public List<Usuario> listarUsuarios() {
        return usuarioService.obtenerTodos();
    }
    
    @GetMapping("/{id}")
    @ResponseBody
    public Usuario obtenerUsuario(@PathVariable Long id) {
        return usuarioService.obtenerPorId(id);
    }
}
```

`@EnableWebMvc` es la anotación que habilita la configuración completa de Spring MVC. Esta anotación registra los componentes necesarios como `HandlerMapping`, `HandlerAdapter`, `ViewResolver` y otros. Cuando se usa Spring Boot, esta configuración se aplica automáticamente a través de `@EnableAutoConfiguration`, por lo que no es necesaria explícitamente a menos que se quiera personalizar la configuración por defecto.

Spring MVC también proporciona soporte para content negotiation, que permite servir el mismo recurso en múltiples formatos (HTML, JSON, XML) basándose en el header `Accept` de la petición. Esta funcionalidad es fundamental para APIs REST que deben soportar múltiples clientes con diferentes necesidades de formato.

El ciclo de vida completo de una petición en Spring MVC incluye la resolución de excepciones, el manejo de multipart requests, la configuración de interceptores y la resolución de argumentos. Cada uno de estos componentes es altamente configurable y extensible, permitiendo adaptar el framework a requisitos específicos de la aplicación.

Los interceptores (HandlerInterceptor) permiten ejecutar lógica antes, durante y después del procesamiento de una petición. Son útiles para tareas como logging, verificación de autenticación, medición de tiempos de respuesta y manipulación de headers. Los interceptores se registran en la configuración de Spring MVC y se aplican a patrones de URL específicos.

```java
@Component
public class TiempoRespuestaInterceptor implements HandlerInterceptor {
    
    private static final Logger log = LoggerFactory.getLogger(TiempoRespuestaInterceptor.class);
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                            Object handler) {
        long inicio = System.currentTimeMillis();
        request.setAttribute("tiempoInicio", inicio);
        return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                          Object handler, ModelAndView modelAndView) {
        long inicio = (long) request.getAttribute("tiempoInicio");
        long duracion = System.currentTimeMillis() - inicio;
        log.info("Tiempo de respuesta para {}: {}ms", 
                request.getRequestURI(), duracion);
    }
}
```

La configuración de Spring MVC se realiza mediante la implementación de `WebMvcConfigurer`. Esta interfaz proporciona métodos para personalizar aspectos como la configuración de contenido, la resolución de argumentos, la configuración de conversores HTTP y la adición de interceptores. Cada método permite modificar un aspecto específico del comportamiento de Spring MVC.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver());
    }
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new TiempoRespuestaInterceptor())
                .addPathPatterns("/api/**");
    }
    
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MappingJackson2HttpMessageConverter());
        converters.add(new StringHttpMessageConverter());
    }
}
```

> Spring MVC no está limitado a aplicaciones web tradicionales. Con la configuración adecuada, puede manejar peticiones WebSocket, Server-Sent Events y otras formas de comunicación bidireccional, manteniendo la misma arquitectura MVC para todos los tipos de comunicación.

