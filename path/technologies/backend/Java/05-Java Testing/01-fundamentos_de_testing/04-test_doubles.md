# Test doubles

El termino test double engloba cualquier objeto que se utiliza en un test en lugar de un objeto real para simular su comportamiento. La metafora proviene del cine: cuando un actor no puede o no debe realizar una escena, se emplea un doble. En testing, cuando el objeto real es lento, no deterministico, dificil de configurar o simplemente no esta disponible, se utiliza un doble. Existen cinco tipos fundamentales, cada uno con un proposito y un nivel de sofisticacion distintos.

## Dummy

Un dummy es el tipo mas simple de test double. Se utiliza exclusivamente para cumplir con la firma de un metodo o constructor, pero nunca se usa realmente en el test. Su unica funcion es permitir que el codigo compile y se ejecute sin lanzar una NullPointerException. Por ejemplo, si un constructor requiere un objeto `Logger`, se puede pasar un dummy (una implementacion vacia de Logger) cuando el comportamiento del logger no es relevante para la prueba.

```java
Logger dummyLogger = new Logger() {
    @Override public void info(String msg) { }
    @Override public void error(String msg) { }
};
var repositorio = new RepositorioUsuario(dummyLogger, baseDeDatosReal);
```

## Stub

Un stub proporciona respuestas predefinidas a las llamadas que recibe durante el test. Su objetivo es forzar al sistema bajo prueba a transitar por un camino especifico. El stub no registra ni verifica las llamadas; simplemente devuelve lo que se le ha programado. Es util cuando se necesita que una dependencia devuelva un valor concreto (un usuario encontrado, una lista vacia, una excepcion) para probar como reacciona el codigo bajo prueba.

```java
when(repositorioStub.buscarPorId(42)).thenReturn(new Usuario("Ana"));
```

## Spy

Un spy envuelve un objeto real y registra las interacciones que ocurren sobre el, permitiendo verificar posteriormente si se llamaron ciertos metodos, con que argumentos y cuantas veces. A diferencia del stub, el spy delega en la implementacion real por defecto, pero expone metadatos de las invocaciones. Es especialmente util en escenarios donde se quiere asegurar que un efecto secundario ocurrio (por ejemplo, que se envio un email o que se persistio un registro), sin perder el comportamiento real del objeto.

```java
var loggerSpy = spy(new LoggerConsola());
servicio.procesar(datos);
verify(loggerSpy).info("Procesamiento completado");
```

## Mock

Un mock es un test double que combina las capacidades de stub y spy, pero con una diferencia filosofica importante: mientras que un test con stubs verifica el estado final del sistema bajo prueba, un test con mocks verifica las interacciones. Es decir, el mock espera que ciertos metodos sean llamados con argumentos especificos, y el test falla si esas interacciones no ocurren. Los mocks se crean tipicamente con librerias como Mockito y se configuran con expectativas que se verifican al final del test. Son la herramienta adecuada cuando lo que interesa es el comportamiento de colaboracion entre objetos.

```java
var emailService = mock(EmailService.class);
servicio.registrarUsuario("user@example.com");
verify(emailService).enviarBienvenida("user@example.com");
```

> La distincion fundamental entre stubs y mocks es el tipo de verificacion: los stubs se usan para probar estado, los mocks para probar interaccion. Usar mocks para todo conduce a tests acoplados a la implementacion.

## Fake

Un fake es una implementacion funcional pero simplificada de una dependencia real que no es adecuada para produccion. A diferencia de stubs y mocks, un fake tiene comportamiento real (logica condicional, estado interno, persistencia), pero de forma ligera. Ejemplos clasicos son una base de datos en memoria (H2 en lugar de PostgreSQL), un sistema de ficheros en memoria o un servicio de email que escribe los mensajes en una lista en lugar de enviarlos realmente. Los fakes son la opcion preferida cuando se necesita probar interacciones complejas con una dependencia sin la sobrecarga de la version real.

```java
// Fake de repositorio en memoria
public class RepositorioUsuarioFake implements RepositorioUsuario {
    private final Map<Long, Usuario> almacen = new HashMap<>();

    @Override
    public Usuario buscarPorId(long id) {
        return almacen.get(id);
    }

    @Override
    public void guardar(Usuario usuario) {
        almacen.put(usuario.getId(), usuario);
    }
}
```

## Cuando usar cada uno

La eleccion del test double adecuado depende del contexto y del objetivo del test. Los dummies cubren requisitos tecnicos sin aportar logica de prueba. Los stubs son ideales para configurar escenarios de entrada y probar la respuesta del sistema. Los fakes proporcionan implementaciones ligeras para pruebas que requieren comportamiento realista. Los mocks y spies se reservan para verificar interacciones, pero deben usarse con moderacion: un uso excesivo de mocks acopla los tests a los detalles de implementacion, haciendolos fragiles frente a refactorizaciones que no cambian el comportamiento observable.
