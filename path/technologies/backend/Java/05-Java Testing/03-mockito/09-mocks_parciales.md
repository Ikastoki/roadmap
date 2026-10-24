# Mocks parciales

Mockito permite crear mocks que preservan el comportamiento real de ciertos metodos mientras reemplazan otros, sin necesidad de usar `@Spy`. Esta tecnica se conoce como mocking parcial y se configura mediante `MockSettings` al crear el mock con `Mockito.mock()`.

El punto de entrada es el metodo `Mockito.withSettings()` que devuelve un objeto `MockSettings` sobre el que se pueden configurar opciones avanzadas del mock:

```java
Calculadora calc = Mockito.mock(Calculadora.class, Mockito.withSettings()
    .defaultAnswer(Mockito.CALLS_REAL_METHODS)
    .name("calculadora-mockeada")
    .verboseLogging()
);
```

La opcion clave es `defaultAnswer(Mockito.CALLS_REAL_METHODS)`. Con esta configuracion, cualquier metodo que no tenga un stub ejecuta su implementacion real. Los metodos stubeados explicitamente con `when()` o `doReturn()` devuelven el valor configurado:

```java
when(calc.sumar(2, 3)).thenReturn(100); // stubbeado: devuelve 100

// Los demas metodos ejecutan implementacion real
int resultado = calc.restar(5, 3);       // real: 2
double resultado2 = calc.dividir(10, 2); // real: 5.0
```

Esta combinacion permite aislar un metodo concreto de una clase sin mockear la clase completa. El resultado es similar a un `@Spy`, pero con la diferencia de que el mock parcial no requiere una instancia preexistente: Mockito crea la instancia internamente usando el constructor sin argumentos (o el que se especifique con `useConstructor()`).

Para afinar el comportamiento, `MockSettings` ofrece opciones adicionales:

```java
Calculadora calc = Mockito.mock(Calculadora.class, Mockito.withSettings()
    .defaultAnswer(Mockito.CALLS_REAL_METHODS)
    .useConstructor(10, 20)           // usa constructor con parametros
    .lenient()                         // modo indulgente (no falla por stubs no usados)
);
```

El modo `lenient()` es util cuando se configuran stubs que no siempre se invocan en todas las pruebas. Sin el modo lenient, Mockito lanza `PotentialStubbingProblem` si un stub configurado no se utiliza durante la ejecucion.

> `CALLS_REAL_METHODS` combinado con clases que tienen efectos secundarios en el constructor puede producir comportamientos inesperados. Usar `withoutAnnotations()` o `useConstructor()` segun corresponda para controlar el proceso de creacion.

El mocking parcial debe usarse con criterio. Si una clase necesita mocking parcial extensivo, probablemente tiene demasiadas responsabilidades o dependencias internas acopladas. La refactorizacion hacia una arquitectura mas desacoplada (extrayendo interfaces, aplicando principio de responsabilidad unica) suele ser una mejor solucion a largo plazo.

Mockito permite que los mocks parciales se combinen con otras opciones como serializacion, invocadores personalizados, o respuestas por defecto especificas:

```java
Repositorio repositorio = Mockito.mock(Repositorio.class, Mockito.withSettings()
    .defaultAnswer(Mockito.CALLS_REAL_METHODS)
    .serializable()
    .stubOnly()
);
```

`stubOnly()` desactiva la verificacion de invocaciones, lo que mejora el rendimiento cuando solo interesa el stubbing y no la verificacion.

La eleccion entre mock parcial (`CALLS_REAL_METHODS`) y espia (`@Spy`) depende del contexto: el espia requiere una instancia real existente o constructor sin argumentos, mientras que el mock parcial permite configurar el constructor y otras opciones a traves de `MockSettings`. Para la mayoria de los casos, `@Spy` resulta mas legible; el mock parcial queda reservado para escenarios que requieren configuraciones avanzadas del mock.
