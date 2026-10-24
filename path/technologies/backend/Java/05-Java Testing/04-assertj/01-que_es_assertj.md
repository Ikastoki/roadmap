# Que es AssertJ

AssertJ es una biblioteca de codigo abierto que proporciona una API fluida y rica para escribir aserciones en pruebas unitarias. Nacio como una evolucion natural de Fest Assert, del cual se bifurco para ofrecer una sintaxis mas expresiva, mensajes de error descriptivos y un modelo de encadenamiento de metodos que elimina la verbosidad tipica de las aserciones tradicionales. Su objetivo es hacer que el codigo de prueba sea no solo correcto, sino tambien legible como lenguaje natural.

## AssertJ vs JUnit assertions

Las aserciones integradas de JUnit, como `assertEquals`, `assertTrue` o `assertNotNull`, son funcionales pero adolecen de varios problemas en proyectos de mediana y gran escala. El primero es la falta de legibilidad: `assertEquals(esperado, actual)` obliga al lector a recordar constantemente que parametro es el esperado y cual es el real. El segundo es la pobreza de los mensajes de error, que a menudo requieren anadir un parametro `String` con una descripcion manual. El tercero es la imposibilidad de encadenar aserciones, lo que lleva a lineas repetitivas y fragmentadas.

AssertJ resuelve estos problemas con un unico punto de entrada: `assertThat(actual)`. A partir de ahi, el metodo retorna un objeto `AbstractAssert` tipado que expone metodos especificos para el tipo del objeto bajo prueba. Un `String` devuelve `StringAssert`, una `List` devuelve `ListAssert`, y asi sucesivamente. Esto permite escribir `assertThat(nombre).isEqualTo("Juan").isNotEmpty().hasSize(4)` en una sola cadena fluida.

```java
// JUnit puro
assertEquals("Juan", usuario.getNombre());
assertNotNull(usuario.getEmail());
assertTrue(usuario.getEdad() > 18);

// AssertJ
assertThat(usuario.getNombre()).isEqualTo("Juan");
assertThat(usuario.getEmail()).isNotNull();
assertThat(usuario.getEdad()).isGreaterThan(18);
```

Los mensajes de error generados por AssertJ incluyen automaticamente la diferencia entre el valor esperado y el real, lo que acelera la depuracion sin necesidad de anadir descripciones manuales.

## Dependencia Maven/Gradle

AssertJ se integra en el proyecto anadiendo una unica dependencia en el ambito de test. La version actual estable se mantiene bajo el grupo `org.assertj`.

```xml
<!-- Maven -->
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.24.2</version>
    <scope>test</scope>
</dependency>
```

```groovy
// Gradle
testImplementation 'org.assertj:assertj-core:3.24.2'
```

> AssertJ no reemplaza el runner de pruebas ni el framework de test (JUnit o TestNG). Se utiliza junto a ellos como biblioteca de aserciones. La combinacion mas habitual es JUnit 5 + AssertJ, donde JUnit gestiona el ciclo de vida de los tests y AssertJ proporciona las aserciones.

Para proyectos que usan Java 8 o superior, la version 3.x es la recomendada. Las versiones 2.x quedaron obsoletas y solo son necesarias para compatibilidad con Java 7. La comunidad de AssertJ mantiene un ritmo de lanzamiento frecuente, por lo que conviene consultar el repositorio oficial para obtener la version mas reciente.
