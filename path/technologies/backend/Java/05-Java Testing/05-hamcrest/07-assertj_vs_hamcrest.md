# AssertJ vs Hamcrest

AssertJ y Hamcrest son las dos bibliotecas de aserciones mas utilizadas en el ecosistema Java. Ambas surgieron para mejorar la legibilidad y expresividad de las pruebas unitarias, pero adoptan enfoques significativamente distintos en su diseno. La eleccion entre una u otra depende del estilo del equipo, la complejidad de las aserciones requeridas y la familiaridad con los paradigmas fluido versus compositivo.

## Diferencias fundamentales

La diferencia arquitectonica principal radica en el modelo de programacion. AssertJ utiliza un modelo fluido basado en el _method chaining_ tipado: cada asercion retorna un objeto asertor especifico para el tipo del valor bajo prueba, y los metodos se encadenan naturalmente.

Hamcrest, en cambio, utiliza un modelo compositivo basado en _matchers_: objetos independientes que se combinan mediante operadores logicos y se pasan como argumento al metodo `assertThat`.

```java
// AssertJ: fluido, tipado, encadenado
assertThat(usuario.getNombre())
    .isNotEmpty()
    .isEqualTo("Ana")
    .startsWith("A");

// Hamcrest: compositivo, funcional, declarativo
assertThat(usuario.getNombre(),
    allOf(not(emptyString()), equalTo("Ana"), startsWith("A")));
```

Cada modelo tiene implicaciones practicas. En AssertJ, el IDE autocompleta los metodos disponibles segun el tipo. En Hamcrest, los matchers se importan estaticamente y se anidan como argumentos.

## AssertJ fluent

El enfoque fluido de AssertJ ofrece varias ventajas: el descubrimiento de metodos mediante el IDE es inmediato (basta escribir `assertThat(valor).` y el autocompletado muestra todas las opciones), los mensajes de error son generados automaticamente por cada asertor con informacion contextual rica, y la sintaxis es lineal: se lee de izquierda a derecha sin necesidad de interpretar anidamientos.

```java
// AssertJ: deteccion automatica del tipo
assertThat(Arrays.asList("a", "b")).contains("a"); // ListAssert
assertThat("texto").startsWith("t");                // StringAssert
assertThat(42).isGreaterThan(10);                  // IntegerAssert
```

El tipado fuerte permite que el compilador valide las aserciones en tiempo de compilacion. No es posible invocar un metodo de `ListAssert` sobre un `StringAssert`, lo que elimina errores de tipo en tiempo de ejecucion.

## Hamcrest combinators

La fortaleza de Hamcrest reside en su sistema de composicion logica. Los matchers se comportan como bloques Lego: cualquier matcher puede combinarse con cualquier otro mediante `allOf`, `anyOf`, `not`, `both` y `either`. Esto permite construir condiciones arbitrariamente complejas sin modificar los matchers existentes.

```java
// Hamcrest: composicion ilimitada
assertThat(valor, allOf(
    not(nullValue()),
    anyOf(
        containsString("ok"),
        containsString("success")
    ),
    startsWith("respuesta_")
));
```

La composicion es especialmente potente para validar estructuras de datos anidadas, donde un matcher puede inspeccionar una propiedad, otro una condicion diferente, y ambos combinarse sin escribir codigo adicional.

## Cuando usar cada uno

AssertJ es la opcion preferida en la mayoria de proyectos nuevos por varias razones: su integracion con IDEs reduce la curva de aprendizaje, su API cubre practicamente todos los tipos del JDK y muchos de terceros (JSON, XML, Optional, Path), no requiere importaciones masivas de matchers, y los mensajes de error son superiores a los de Hamcrest por defecto.

Hamcrest sigue siendo relevante en proyectos donde la composicion logica es un requisito frecuente, en equipos acostumbrados a su sintaxis declarativa, o en proyectos que ya usan Hamcrest y no justifican la migracion. Tambien es la eleccion natural en entornos donde se valora la independencia de los matchers como ciudadanos de primera clase, por ejemplo para compartir matchers personalizados entre multiples proyectos.

Como regla general: si las aserciones son mayoritariamente directas (comparaciones, nulidad, tamaño), AssertJ ofrece una sintaxis mas limpia. Si se necesita componer condiciones logicas complejas con frecuencia, Hamcrest puede resultar mas expresivo.

## Combinarlos

Es perfectamente posible usar AssertJ y Hamcrest en el mismo proyecto, siempre que se gestione correctamente la importacion estatica para evitar conflictos con el metodo `assertThat`. La estrategia consiste en usar el nombre completo o alias para una de las dos bibliotecas:

```java
// Usar AssertJ como principal, Hamcrest para casos puntuales
import static org.assertj.core.api.Assertions.assertThat;
import org.hamcrest.Matchers;

@Test
void testCombinado() {
    String valor = "resultado_ok_42";

    // AssertJ para asercion directa
    assertThat(valor).startsWith("resultado");

    // Hamcrest para composicion logica especifica
    org.hamcrest.MatcherAssert.assertThat(valor,
        Matchers.allOf(
            Matchers.containsString("ok"),
            Matchers.containsString("42")
        ));
}
```

Otra alternativa es usar Hamcrest solo en aserciones que requieran composicion y delegar el resto a AssertJ. Algunos equipos incluso crean una fachada unificada que expone metodos de ambas bibliotecas con nombres diferenciados.

> Mezclar bibliotecas de aserciones en un mismo proyecto no es intrinsecamente malo, pero debe hacerse con criterio. La inconsistencia en el estilo de las aserciones dentro de un mismo equipo reduce la legibilidad del codigo. Lo recomendable es elegir una biblioteca como opcion por defecto y reservar la otra para casos muy especificos que la biblioteca principal no cubra bien.

```java
// Resumen comparativo
// AssertJ: assertThat(valor).metodo().otroMetodo()
// Hamcrest: assertThat(valor, matcher)
```
