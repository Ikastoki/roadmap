# Pointcut Expressions

Las expresiones de pointcut son el mecanismo mediante el cual Spring AOP identifica los join points sobre los cuales actuan los aspectos. Cada expresion describe un patron que puede coincidir con uno o multiples metodos de la aplicacion. Spring AOP soporta un subconjunto del lenguaje de pointcut de AspectJ, suficiente para la mayoria de casos de uso que involucran metodos publicos de beans gestionados por el contenedor.

Una expresion de pointcut se compone de un designador (el tipo de match), un patron de coincidencia (la ruta del metodo o clase) y opcionalmente una referencia a tipos o anotaciones. La sintaxis sigue el formato `designador(patrones)`, donde los patrones utilizan wildcards y expresiones regulares simplificadas.

## Designador execution

El designador `execution` es el mas utilizado y flexible. Matchea metodos segun su firma completa: modificador de acceso, tipo de retorno, paquete, clase, nombre del metodo y tipos de argumentos. La sintaxis general es:

```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)
```

Los signos de interrogacion indican componentes opcionales. Si se omite un componente, cualquier valor coincide para esa parte de la firma.

**Coincidencia por paquete y clase:**

```java
// Todos los metodos publicos de cualquier clase en com.miapp.service
@Pointcut("execution(* com.miapp.service.*.*(..))")
public void todosServicios() {}

// Todos los metodos (cualquier modificador) de UserServiceImpl
@Pointcut("execution(* com.miapp.service.impl.UserServiceImpl.*(..))")
public void metodosUserServiceImpl() {}
```

**Coincidencia por nombre de metodo:**

```java
// Cualquier metodo que empiece por "save" en el paquete service
@Pointcut("execution(* com.miapp.service.*.save*(..))")
public void metodosSave() {}

// Metodos que contengan "find" en cualquier clase del paquete
@Pointcut("execution(* com.miapp.service.*.*find*(..))")
public void metodosFind() {}
```

**Coincidencia por tipo de retorno:**

```java
// Solo metodos que retornan un objeto de tipo List
@Pointcut("execution(java.util.List com.miapp.service.*.*(..))")
public void metodosQueRetornanList() {}

// Metodos que retornan void
@Pointcut("execution(void com.miapp.service.*.*(..))")
public void metodosVoid() {}
```

**Coincidencia por argumentos:**

```java
// Metodos que reciben exactamente dos argumentos String
@Pointcut("execution(* com.miapp.service.*.*(String, String))")
public void dosStrings() {}

// Metodos que reciben un Long como primer argumento y cualquier cosa despues
@Pointcut("execution(* com.miapp.service.*.*(Long, ..))")
public void empiezaConLong() {}

// Metodos sin argumentos
@Pointcut("execution(* com.miapp.service.*.*())")
public void sinArgumentos() {}
```

El patron `(..)` indica cero o mas argumentos de cualquier tipo. Un patron como `(String, ..)` indica que el primer argumento es un String, seguido de cero o mas argumentos adicionales.

## Designador within

El designador `within` restringe la coincidencia a metodos dentro de una clase o paquete determinado. A diferencia de `execution`, `within` no permite filtrar por firma del metodo, solo por la ubicacion de la clase.

```java
// Todos los metodos dentro de clases del paquete com.miapp.service
@Pointcut("within(com.miapp.service..*)")
public void dentroPaqueteService() {}

// Todos los metodos dentro de una clase especifica
@Pointcut("within(com.miapp.service.UserService)")
public void dentroUserService() {}
```

> La diferencia entre `execution(* com.miapp.service.*.*(..))` y `within(com.miapp.service.*)` es que `execution` matchea metodos de clases que pueden estar en subpaquetes segun la semantica del designador, mientras que `within` es estricto con la ruta del paquete. En la practica, `execution` es preferido por su flexibilidad.

## Designador @annotation

El designador `@annotation` matchea metodos que poseen una anotacion especifica. Es particularmente util para implementar comportamiento transversal basado en marcadores personalizados, como una anotacion `@Auditable` que marque los metodos que requieren registro de auditoria.

```java
// Definicion de la anotacion personalizada
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Auditable {
    String descripcion() default "";
}
```

```java
@Aspect
@Component
public class AuditoriaAspect {

    @Before("@annotation(auditable)")
    public void auditar(JoinPoint joinPoint, Auditable auditable) {
        String desc = auditable.descripcion();
        System.out.println("[AUDIT] " + joinPoint.getSignature().getName() 
            + " - " + desc);
    }
}
```

```java
@Service
public class PedidoService {

    @Auditable(descripcion = "Creacion de pedido")
    public Pedido crearPedido(Pedido pedido) {
        // logica de negocio
        return pedido;
    }

    @Auditable
    public void cancelarPedido(Long id) {
        // logica de negocio
    }
}
```

El parametro del designador (`auditable`) se inyecta automaticamente en el advice, permitiendo acceder a los atributos de la anotacion sin necesidad de reflexion explicita.

## Designador args

El designador `args` permite filtrar join points segun los tipos de los argumentos del metodo. A diferencia de `execution`, que combina patron de nombre con tipos de argumento, `args` se enfoca exclusivamente en los argumentos y puede bindear valores para usarlos en el advice.

```java
// Metodos que reciben un objeto de tipo Pedido como primer argumento
@Pointcut("execution(* com.miapp.service.*.*(..)) && args(pedido,..)")
public void conArgumentoPedido(Pedido pedido) {}

@AfterReturning(pointcut = "conArgumentoPedido(pedido)", returning = "resultado")
public void despuesDePedido(JoinPoint joinPoint, Pedido pedido, Object resultado) {
    System.out.println("Pedido procesado: " + pedido.getId());
}
```

El binding de argumentos con `args()` en el pointcut permite que el parametro se pase directamente al advice, eliminando la necesidad de extraerlo del array de `getArgs()`.

## Designador bean

El designador `bean` es una extension de Spring AOP (no disponible en AspectJ puro) que permite restringir la coinccurrencia al nombre del bean del contenedor. Es util cuando se necesita aplicar un aspecto unicamente a beans especificos sin importar en que paquete o clase estan definidos.

```java
// Solo metodos del bean llamado "userService"
@Pointcut("execution(* *.*(..)) && bean(userService)")
public void soloUserService() {}

// Beans que coinciden con un patron
@Pointcut("execution(* *.*(..)) && bean(*Service)")
public void beansService() {}
```

## Combinacion de Pointcuts

Spring AOP permite combinar expresiones de pointcut usando operadores logicos `&&` (AND), `||` (OR) y `!` (NOT). Esto permite construir expresiones complejas a partir de expresiones simples.

```java
// Interseccion: ambos condiciones deben cumplirse
@Pointcut("execution(* com.miapp.service.*.*(..)) && @annotation(auditable)")
public void metodosAuditableEnService() {}

// Union: cualquiera de las condiciones es suficiente
@Pointcut("execution(* com.miapp.service.*.save*(..)) || execution(* com.miapp.service.*.delete*((..))")
public void operacionesDeEscritura() {}

// Negacion: excluye metodos marcados con la anotacion
@Pointcut("execution(* com.miapp.service.*.*(..)) && !@annotation(SkipAudit)")
public void metodosSinSkipAudit() {}
```

Cuando un pointcut con nombre se define en una clase diferente, se referencia usando el prefijo del fully qualified name:

```java
@Aspect
@Component
public class CrossCuttingAspect {

    @Before("com.miapp.aspects.CommonPointcuts.todosServicios()")
    public void antesDelServicio(JoinPoint joinPoint) {
        // ...
    }
}
```

> Los operadores `&&`, `||` y `!` permiten construir expresiones arbitrariamente complejas. Sin embargo, en la practica es preferible definir pointcuts con nombre claros y composicion sencilla, ya que expresiones complejas dificultan el mantenimiento.

