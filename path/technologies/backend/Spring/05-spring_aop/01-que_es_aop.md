# Que es AOP

Aspect-Oriented Programming (AOP) es un paradigma de programacion que complementa a la Programacion Orientada a Objetos (OOP) introduciendo la capacidad de modularizar funcionalidades transversales. Mientras que OOP organiza el codigo en torno a entidades del dominio (clases y objetos), AOP permite separar preocupaciones que cruzan multiples modulos sin acoplar el codigo principal con logica auxiliar como logging, seguridad, transacciones o manejo de excepciones.

En cualquier aplicacion de cierto tamano, existen responsabilidades que no pertenecen a un unico modulo pero que deben aplicarse de forma uniforme. Un ejemplo clasico es el registro de llamadas a metodos de servicio: sin AOP, cada metodo de negocio deberia contener lineas de logging repetitivas, violando el principio DRY y ensuciando la logica de negocio con preocupaciones instrumentales. AOP resuelve esto definiendo esa logica una sola vez como un aspecto y declarando donde debe aplicarse mediante puntos de corte.

## Cross-Cutting Concerns

Las cross-cutting concerns son funcionalidades que afectan a multiples puntos de la aplicacion sin que exista un lugar natural en la jerarquia de clases para colocarlas. Algunas de las mas comunes incluyen logging, auditoria, seguridad (autenticacion y autorizacion), manejo de excepciones, caching, transacciones y metricas de rendimiento. Estas preocupaciones "cruzan" los modulos de negocio de forma que un enfoque puramente OOP genera duplicacion o acoplamiento indeseado.

Cuando una cross-cutting concern se distribuye por toda la base de codigo, mantenerla o modificarla implica recorrer multiples archivos. Si se decide cambiar el formato de los logs de auditoria o agregar un nivel de verificacion de permisos, el impacto puede ser amplio y propenso a errores. AOP centraliza esa logica en un solo lugar, el aspecto, y delega al framework la responsabilidad de inyectarla en los puntos adecuados.

## AOP vs OOP

La diferencia fundamental entre ambos paradigmas no es de sustitucion sino de complemento. OOP modela el dominio del problema mediante herencia, polimorfismo y encapsulamiento. AOP modela el comportamiento transversal mediante aspectos que se aplican sobre el codigo existente. Un servicio de negocio sigue siendo una clase con metodos que implementan reglas de negocio; el aspecto se encarga de inyectar antes o despues de esas llamadas la logica横 transversal.

En terminos de arquitectura, OOP opera en el eje vertical (jerarquia de clases y dependencias entre modulos) mientras que AOP opera en el eje horizontal (comportamientos que atraviesan multiples capas). Esta separacion permite que el desarrollador de negocio no necesite conocer detalles de instrumentacion, seguridad o auditoria, y que el especialista en seguridad o observabilidad defina los aspectos de forma independiente.

## Terminologia AOP

La terminologia de AOP tiene sus propios conceptos que conviene dominar antes de profundizar en Spring AOP:

**Aspecto (Aspect):** Un modulo que encapsula una cross-cutting concern. En Spring se define usando la anotacion `@Aspect` sobre una clase Java. El aspecto contiene tanto la definicion de donde se aplica (pointcuts) como la logica que se ejecuta (advice).

**Advice (Consejo):** La accion que se ejecuta en un punto determinado del programa. Es el "que" del aspecto: el codigo que se invoca. Puede ejecutarse antes, despues, alrededor o en combinacion con el metodo interceptado.

**Pointcut (Punto de corte):** Una expresion que define donde se aplica el advice. Determina sobre cuales metodos, clases o paquetes actua el aspecto. Un pointcut bien definido permite reutilizar el mismo advice en multiples ubicaciones.

**Join Point (Punto de union):** Un punto especifico en la ejecucion del programa donde se puede insertar un advice. En Spring AOP, los join points son siempre la ejecucion de metodos publicos de un bean gestionado por el contenedor.

**Weaving (Tejido):** El proceso de conectar los aspectos con los objetos objetivo para crear objetos proxy. Es el momento en que Spring combina el advice con los pointcuts para generar el comportamiento transversal.

## Tipos de Weaving

El weaving puede realizarse en distintas etapas del ciclo de vida de la aplicacion, y cada estrategia tiene implicaciones en rendimiento, flexibilidad y complejidad de configuracion.

**Compile-time weaving:** Los aspectos se aplican durante la compilacion del codigo fuente. El compilador (o un post-procesador) modifica los archivos `.class` insertando la logica del advice directamente en los metodos que coinciden con los pointcuts. AspectJ es la herramienta de referencia para este modo. Produce proxies sin overhead en tiempo de ejecution pero requiere herramientas de compilacion especiales y no funciona con proxies dinamicos.

**Load-time weaving (LTW):** Los aspectos se aplican durante la carga de clases por el ClassLoader. La definicion de aspectos se proporciona en archivos de configuracion o anotaciones y el agente de JVM (javaagent) modifica las clases justo antes de que se carguen en memoria. Ofrece un equilibrio entre compile-time y runtime weaving: no necesita compilacion especial pero la carga inicial es mas lenta.

**Runtime weaving:** Los aspectos se aplican en tiempo de ejecution creando proxies dinamicos. Spring AOP opera en este modo por defecto. Utiliza JDK Dynamic Proxy para interfaces y CGLIB para clases concretas. Es el modo mas flexible y compatible con cualquier IDE y herramienta de compilacion estandar, aunque introduce un nivel minimo de indireccion.

> En la practica, el 90% de los proyectos con Spring AOP utilizan runtime weaving con proxies CGLIB. Solo se recurre a AspectJ (compile-time o LTW) cuando se necesita interceptar construcciones, campos o metodos privados, que estan fuera del alcance de los proxies dinamicos.

## AspectJ y Spring AOP

AspectJ es la implementacion de referencia del estandar AOP para Java. Proporciona un lenguaje de pointcut rico, soporte completo para todos los join points (incluidos constructores y campos) y weaving en compile-time o load-time. Spring AOP, por su parte, es una implementacion simplificada que opera exclusivamente en runtime mediante proxies y limita los join points a la ejecucion de metodos publicos.

La decision de usar Spring AOP o AspectJ depende del alcance necesario. Si la instrumentacion afecta unicamente a metodos publicos de beans gestionados por el contenedor, Spring AOP es suficiente y no requiere configuracion adicional. Si se necesita interceptar metodos internos, constructores, campos o clases no gestionadas por Spring, AspectJ es la herramienta adecuada.

```java
// Spring AOP: solo intercepta metodos publicos de beans del contenedor
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.miapp.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Llamando a: " + joinPoint.getSignature().getName());
    }
}
```

```java
// AspectJ: puede interceptar cualquier punto de ejecucion, incluso privados
public aspect LoggingAspect {
    before(): execution(* com.miapp.service.*.*(..)) {
        System.out.println("Llamando a: " + thisJoinPoint.getSignature().getName());
    }
}
```

> La simplificacion de Spring AOP frente a AspectJ es intencional: al limitar los join points a metodos publicos de beans gestionados, se elimina la necesidad de bytecode manipulation y se mantiene la compatibilidad con proxies simples.

