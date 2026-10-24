# AOP Proxies

El mecanismo central de Spring AOP es la creacion de proxies: objetos que envuelven al bean real e interceptan las llamadas a sus metodos para inyectar el advice de los aspectos. Spring AOP opera exclusivamente en tiempo de ejecution mediante proxies dinamicos, lo que significa que los aspectos se aplican cuando el contenedor crea los beans, sin necesidad de modificar el bytecode o recompilar el codigo.

Cada bean gestionado por el contenedor que es afectado por al menos un aspecto se envuelve en un proxy. Cuando el resto de la aplicacion (controllers, repositories, otros servicios) realiza una llamada a un metodo de ese bean, en realidad esta invocando al proxy, que antes de delegar al objeto real ejecuta el advice correspondiente. Para el caller, el proxy es transparente: implementa la misma interfaz o hereda de la misma clase.

## JDK Dynamic Proxy

JDK Dynamic Proxy es la capacidad nativa de Java para crear proxies en tiempo de ejecution de interfaces. Requiere que el objeto a proxificar implemente al menos una interfaz y genera una clase proxy que implementa las mismas interfaces. El proxy intercepta todas las invocaciones de metodos y las delega a un `InvocationHandler`.

```java
// Ejemplo conceptual de como funciona JDK Dynamic Proxy
interface UserService {
    Usuario findByUsername(String username);
}

public class UserServiceImpl implements UserService {
    public Usuario findByUsername(String username) {
        return repository.findByUsername(username);
    }
}

// Spring crea internamente algo equivalente a:
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class[]{UserService.class},
    (proxyObj, method, args) -> {
        System.out.println("Antes de: " + method.getName());
        Object resultado = method.invoke(userServiceImpl, args);
        System.out.println("Despues de: " + method.getName());
        return resultado;
    }
);
```

El `InvocationHandler` recibe cada llamada al proxy y decide que hacer: ejecutar logica antes, despues o alrededor de la invocacion real, o incluso bloquear la llamada por completo. Spring registra automaticamente un handler que recorre la cadena de asesores (advisors) asociados al bean y ejecuta el advice correspondiente.

La limitacion principal de JDK Dynamic Proxy es que solo funciona con interfaces. Si un bean no implementa ninguna interfaz, no puede ser proxificado de esta forma. Ademas, si el caller invoca un metodo definido en la clase concreta pero no en la interfaz, el proxy no puede interceptarlo.

## CGLIB Proxy

CGLIB (Code Generation Library) es una libreria que genera subclases en tiempo de ejecution mediante manipulacion de bytecode. A diferencia de JDK Dynamic Proxy, CGLIB no requiere interfaces: crea una subclase del bean real y sobreescribe sus metodos para interceptar las llamadas. El proxy extiende la clase original y mantiene la herencia intacta.

```java
// Ejemplo conceptual de como funciona CGLIB proxy
// Spring genera internamente algo equivalente a:
class UserServiceImpl_CGLIB_Proxy extends UserServiceImpl {
    private MethodInterceptor interceptor;

    @Override
    public Usuario findByUsername(String username) {
        // Interceptar y ejecutar advice
        return interceptor.intercept(this, findMethod, new Object[]{username}, 
            new MethodProxy(userServiceImpl, findMethod));
    }
}

// El proxy se crea asi:
UserService proxy = (UserService) new Enhancer()
    .setSuperclass(UserServiceImpl.class)
    .setCallback(handler)
    .create();
```

CGLIB genera una subclase donde cada metodo sobrescrito delega al `MethodInterceptor` antes de invocar al metodo de la superclase. Esto permite interceptar metodos concretos incluso cuando no existen interfaces, y funciona con herencia, polimorfismo y cualquier tipo de retorno.

La desventaja de CGLIB es que no puede interceptar metodos `final` ni clases `final`, ya que genera subclases. Tampoco puede interceptar metodos estaticos. Estas restricciones son raramente un problema en la practica, ya que Spring desaconseja el uso de `final` en beans por otras razones (como la inyeccion de dependencias por proxy).

## Configuracion del Tipo de Proxy

Spring Boot, a partir de la version 2.0, configura por defecto CGLIB proxying con la propiedad `spring.aop.proxy-target-class=true`. Esto significa que todos los beans se proxifican con CGLIB independientemente de si implementan interfaces o no, comportamiento consistente y predecible.

En Spring Framework sin Boot, el comportamiento por defecto varia segun la version. Antes de Spring 5, la estrategia por defecto era JDK Dynamic Proxy si el bean implementaba interfaces. Spring 5+ cambio el default a CGLIB para ser consistente con Spring Boot.

```properties
# Spring Boot 2.x+ - CGLIB por defecto (no es necesario configurar)
spring.aop.proxy-target-class=true

# Para forzar JDK Dynamic Proxy (si se requiere por compatibilidad)
spring.aop.proxy-target-class=false
```

Para configurar manualmente el tipo de proxy en XML (aplicaciones no-Boot):

```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

O en configuracion Java:

```java
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class AopConfig {
}
```

> Si se fuerza JDK Dynamic Proxy y un bean no implementa ninguna interfaz, Spring lanzara una `BeanCreationException` indicando que no es posible crear el proxy. En ese caso, se debe volver a CGLIB o crear una interfaz para el bean.

## Forced CGLIB Proxying

En ciertos escenarios es necesario forzar CGLIB proxying aunque se tengan interfaces. Un caso comun es cuando se necesita inyectar un proxy de la clase concreta en lugar de la interfaz, o cuando se requiere que metodos de la implementacion (no de la interfaz) sean interceptados.

```java
@Service
public class NotificationService {
    
    public void enviarNotificacion(Notificacion notif) {
        // este metodo solo existe en la clase concreta, no en ninguna interfaz
        // CGLIB puede interceptarlo
    }
}
```

Con `proxy-target-class=true`, Spring siempre crea un proxy CGLIB que hereda de `NotificationService`. El proxy tiene todos los metodos de la clase, tanto los definidos en interfaces como los propios de la implementacion.

## Limitaciones de los Proxies

Los proxies de Spring AOP tienen limitaciones inherentes al mecanismo de proxying. Primero, solo interceptan metodos invocados desde fuera del objeto proxy. Si un metodo del propio bean llama a otro metodo del mismo bean, la llamada interna bypassa el proxy y el advice no se ejecuta. Este es el problema de la autoinvocacion (self-invocation).

```java
@Service
public class OrderService {

    @Transactional
    public void crearPedido(Pedido pedido) {
        // Esta llamada NO passa por el proxy
        validarPedido(pedido); 
        repository.save(pedido);
    }

    @Transactional(readOnly = true)
    public void validarPedido(Pedido pedido) {
        // El advice @Transactional de este metodo NO se ejecuta
        // cuando es llamado desde crearPedido()
    }
}
```

Segundo, los proxies no pueden interceptar metodos `final`, `static` ni constructores. Tercero, si se inyecta el bean por su tipo concreto y se asigna a una variable del mismo tipo (no de interfaz), el proxy se "desenvuelve" y las llamadas posteriores bypassan el proxy.

> El problema de autoinvocacion se resuelve extrayendo el metodo que necesita ser proxificado a otro bean, inyectandolo y llamandolo desde ahi, o utilizando `AopContext.currentProxy()` para obtener el proxy actual (requiere `exposeProxy=true`).

