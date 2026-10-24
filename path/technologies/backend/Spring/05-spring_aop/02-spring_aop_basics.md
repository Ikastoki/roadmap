# Spring AOP Basics

Spring AOP implementa el paradigma de programacion orientada a aspectos utilizando proxies dinamicos en tiempo de ejecution. Para definir aspectos en Spring se emplea la anotacion `@Aspect` combinada con anotaciones de advice que determinan el momento en que se ejecuta la logica transversal respecto al metodo interceptado. El contenedor Spring detecta automaticamente las clases anotadas con `@Aspect` y las registra como asesores en el contexto de la aplicacion.

## Anotacion @Aspect

Una clase marcada con `@Aspect` se convierte en un modulo de AOP que contiene pointcuts y advice. Para que Spring la detecte, ademas de `@Aspect` debe estar registrada como bean, ya sea mediante `@Component` o mediante algun stereotype de la familia `@Service`, `@Repository`, etc. Spring Boot configura automaticamente la habilitacion de AOP cuando detecta `spring-aop` en el classpath.

```java
@Aspect
@Component
public class AuditoriaAspect {

    @Before("execution(* com.miapp.service.*.*(..))")
    public void auditarMetodo(JoinPoint joinPoint) {
        String metodo = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        System.out.println("[AUDITORIA] Metodo: " + metodo + " | Args: " + args);
    }
}
```

> En Spring Boot, si `spring-boot-starter-aop` esta en las dependencias, AOP se habilita automaticamente. No es necesario agregar `@EnableAspectJAutoProxy` salvo que se necesiten configuraciones avanzadas de proxying.

## Anotacion @Pointcut

Un pointcut define un conjunto de join points donde se aplican los advice. Spring permite definir pointcuts con nombre reutilizables, lo que evita duplicar expresiones en cada advice. Un metodo anotado con `@Pointcut` actua como referencia para ser invocado en las anotaciones de advice.

```java
@Aspect
@Component
public class PerformanceAspect {

    @Pointcut("execution(* com.miapp.service.*.*(..))")
    public void metodosServicio() {
        // El cuerpo del metodo se deja vacio; el pointcut se define por la anotacion
    }

    @Around("metodosServicio()")
    public Object medirTiempo(ProceedingJoinPoint joinPoint) throws Throwable {
        long inicio = System.currentTimeMillis();
        Object resultado = joinPoint.proceed();
        long duracion = System.currentTimeMillis() - inicio;
        System.out.println("[PERFORMANCE] " + joinPoint.getSignature().getName() 
            + " ejecutado en " + duracion + " ms");
        return resultado;
    }
}
```

El nombre del metodo (`metodosServicio()`) se utiliza como identificador en las expresiones de advice. Esto permite mantener la expresion de pointcut en un solo lugar y reutilizarla en multiples advice dentro del mismo aspecto o en otros aspectos referenciando el punto de corte por nombre.

## Tipos de Advice

Spring AOP ofrece cinco tipos de advice que determinan el momento de ejecucion del comportamiento transversal. Cada tipo tiene implicaciones distintas en cuanto a acceso al flujo de ejecucion, capacidad de modificacion del retorno y manejo de excepciones.

### @Before

El advice `@Before` se ejecuta antes de que el metodo objetivo invocado por el join point comience su ejecucion. No puede modificar el retorno del metodo ni prevenir que se ejecute. Si el advice lanza una excepcion, el metodo objetivo no se ejecuta y la excepcion se propaga al caller.

```java
@Aspect
@Component
public class SecurityAspect {

    @Before("execution(* com.miapp.service.admin.*.*(..))")
    public void verificarPermisos(JoinPoint joinPoint) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            throw new AccessDeniedException("Usuario no autenticado");
        }
        System.out.println("[SECURITY] Acceso autorizado para: " + auth.getName());
    }
}
```

### @After

El advice `@After` se ejecuta despues de la invocacion del metodo objetivo, independientemente de si el metodo termino con exito o lanzo una excepcion. Equivalente a un bloque `finally` en Java. No puede acceder al valor de retorno ni a la excepcion lanzada.

```java
@Aspect
@Component
public class ResourceAspect {

    @After("execution(* com.miapp.service.*.*(..))")
    public void liberarRecursos(JoinPoint joinPoint) {
        System.out.println("[RESOURCE] Recursos liberados despues de: " 
            + joinPoint.getSignature().getName());
    }
}
```

### @AfterReturning

El advice `@AfterReturning` se ejecuta unicamente cuando el metodo objetivo retorna sin lanzar excepciones. Permite acceder al valor de retorno y potencialmente modificarlo (en Spring 5.x y posteriores). El atributo `returning` especifica el nombre del parametro que recibira el valor de retorno.

```java
@Aspect
@Component
public class CacheAspect {

    @AfterReturning(pointcut = "execution(* com.miapp.service.*.*(..))", returning = "resultado")
    public void almacenarEnCache(JoinPoint joinPoint, Object resultado) {
        String clave = joinPoint.getSignature().getName() + Arrays.toString(joinPoint.getArgs());
        cache.put(clave, resultado);
        System.out.println("[CACHE] Almacenado en cache: " + clave);
    }
}
```

> El parametro que recibe el valor de retorno en `@AfterReturning` es de tipo `Object` por defecto. Se puede especificar un tipo concreto en el pointcut o en el parametro para evitar casting manual.

### @AfterThrowing

El advice `@AfterThrowing` se ejecuta solo cuando el metodo objetivo lanza una excepcion. Permite inspeccionar la excepcion lanzada pero no prevenir su propagacion. El atributo `throwing` define el nombre del parametro que contiene la excepcion.

```java
@Aspect
@Component
public class ErrorHandlingAspect {

    @AfterThrowing(pointcut = "execution(* com.miapp.service.*.*(..))", throwing = "ex")
    public void manejarExcepcion(JoinPoint joinPoint, Exception ex) {
        System.err.println("[ERROR] Excepcion en " + joinPoint.getSignature().getName() 
            + ": " + ex.getMessage());
    }
}
```

### @Around

El advice `@Around` es el mas poderoso de los cinco tipos. Envuelve completamente la ejecucion del metodo objetivo, teniendo control total sobre si este se ejecuta o no, y sobre el valor que retorna. Recibe un `ProceedingJoinPoint` en lugar de `JoinPoint`, lo que permite invocar explicitamente al metodo objetivo con `joinPoint.proceed()`.

```java
@Aspect
@Component
public class TransactionAspect {

    @Around("execution(* com.miapp.service.*.*(..))")
    public Object manejarTransaccion(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("[TX] Iniciando transaccion...");
        try {
            Object resultado = joinPoint.proceed();
            System.out.println("[TX] Commit de transaccion");
            return resultado;
        } catch (Exception e) {
            System.out.println("[TX] Rollback de transaccion");
            throw e;
        }
    }
}
```

Un aspecto `@Around` puede modificar el argumento que se pasa al metodo objetivo, cambiar el objeto de retorno, suprimir la excepcion o incluso no invocar al metodo en absoluto. Esta flexibilidad lo convierte en el advice preferido cuando se necesita control granular sobre la ejecucion.

> Si un `@Around` advice no invoca `joinPoint.proceed()`, el metodo objetivo no se ejecutara. Esto es util para implementar circuit breakers o validaciones previas que pueden bloquear la ejecucion completa.

## JoinPoint vs ProceedingJoinPoint

`JoinPoint` es la interfaz que proporciona informacion sobre el punto de ejecucion interceptado. Ofrece metodos para obtener la firma del metodo (`getSignature()`), los argumentos (`getArgs()`), el objeto objetivo (`getTarget()`) y el proxy (`getThis()`). Es el tipo de parametro que reciben `@Before`, `@After`, `@AfterReturning` y `@AfterThrowing`.

`ProceedingJoinPoint` extiende `JoinPoint` y agrega el metodo `proceed()` que permite invocar al metodo objetivo. Solo esta disponible en advice `@Around` porque es el unico tipo que necesita control explicito sobre la continuacion de la ejecucion.

```java
@Around("metodosServicio()")
public Object control(ProceedingJoinPoint pjp) throws Throwable {
    // Informacion del join point
    Signature firma = pjp.getSignature();
    Object[] args = pjp.getArgs();
    
    // El proceed() invoca al metodo real
    // Se puede pasar argumentos modificados
    return pjp.proceed(args);
}
```

## Orden de Aspectos con @Order

Cuando multiples aspectos interceptan el mismo join point, el orden de ejecucion se determina con la anotacion `@Order`. El valor numerico indica la precedencia: un valor menor implica mayor prioridad. Los aspectos de mayor prioridad se ejecutan primero en el advice `@Before` y ultimo en el advice `@After`, creando un efecto de "cebolla" donde los aspectos se apilan externamente.

```java
@Aspect
@Component
@Order(1)
public class PrimerAspecto {
    @Before("execution(* com.miapp.service.*.*(..))")
    public void antes() {
        System.out.println("1. Primer aspecto - ANTES");
    }

    @After("execution(* com.miapp.service.*.*(..))")
    public void despues() {
        System.out.println("1. Primer aspecto - DESPUES");
    }
}

@Aspect
@Component
@Order(2)
public class SegundoAspecto {
    @Before("execution(* com.miapp.service.*.*(..))")
    public void antes() {
        System.out.println("2. Segundo aspecto - ANTES");
    }

    @After("execution(* com.miapp.service.*.*(..))")
    public void despues() {
        System.out.println("2. Segundo aspecto - DESPUES");
    }
}
```

Para una llamada a un metodo de servicio, la salida seria:

```
1. Primer aspecto - ANTES
2. Segundo aspecto - ANTES
[metodo real se ejecuta]
2. Segundo aspecto - DESPUES
1. Primer aspecto - DESPUES
```

Este patron de ordenamiento es identico al de los filtros Servlet o los interceptores de Spring MVC: el aspecto con menor `@Order` envuelve a los demas como la capa externa de una cebolla.

