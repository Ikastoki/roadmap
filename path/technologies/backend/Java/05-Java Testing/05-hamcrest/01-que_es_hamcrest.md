# Que es Hamcrest

Hamcrest es una biblioteca de _matchers_ (coincidentes) que permite escribir aserciones declarativas y legibles en pruebas unitarias. Su nombre es un anagrama de _matchers_, y su filosofia central es tratar las aserciones como una composicion de pequenos predicados reutilizables, combinables mediante operadores logicos. Originalmente concebida para Java, Hamcrest ha sido portada a multiples lenguajes incluyendo Python, JavaScript, Ruby y Objective-C.

## Library de matchers, legibilidad y composicion

La idea fundamental de Hamcrest es que una asercion se construye como una oracion: el metodo `assertThat` recibe un valor real y un _matcher_ que describe la condicion esperada. El resultado es una expresion que se lee casi como lenguaje natural.

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

assertThat("Hola Mundo", containsString("Mundo"));
assertThat(42, is(greaterThan(10)));
assertThat(Arrays.asList("a", "b"), hasItem("a"));
```

Cada matcher es un objeto independiente que implementa la interfaz `Matcher<T>`. Esto permite componerlos: un matcher puede contener a otro, negarlo, combinarlo con condiciones logicas o aplicarlo a colecciones completas. La composicion es la clave de la expresividad de Hamcrest.

```java
assertThat("Hola", anyOf(containsString("Hol"), containsString("Adios")));
assertThat(Arrays.asList("a", "b", "c"), everyItem(not(emptyString())));
```

A diferencia de las aserciones tradicionales donde cada metodo encapsula una verificacion atomica (por ejemplo `assertEquals`, `assertTrue`), Hamcrest separa la accion de aseverar (`assertThat`) del predicado que se verifica (el matcher). Esto permite reutilizar los mismos matchers en distintos contextos y combinarlos de formas que el disenador original no anticipo.

> Hamcrest no sustituye a JUnit ni a TestNG. Se integra con ellos como una biblioteca de aserciones independiente. De hecho, JUnit 4 incluye Hamcrest como dependencia interna para su propio `assertThat`, aunque JUnit 5 elimino esa dependencia directa y recomienda incluir Hamcrest explicitamente si se desea usar.

## Dependencia Maven/Gradle

Hamcrest se distribuye en varios modulos. El nucleo (`hamcrest-core`) contiene los matchers basicos y la interfaz `Matcher`. El modulo `hamcrest-library` anade matchers para colecciones, numeros, textos, objetos y tipos personalizados. A partir de la version 2.x, Hamcrest unifico su empaquetado bajo el grupo `org.hamcrest`.

```xml
<!-- Maven -->
<dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest</artifactId>
    <version>2.2</version>
    <scope>test</scope>
</dependency>
```

```groovy
// Gradle
testImplementation 'org.hamcrest:hamcrest:2.2'
```

> En versiones anteriores, Hamcrest se distribuia en `hamcrest-core` y `hamcrest-library` por separado. A partir de 2.1, el artefacto `hamcrest` incluye ambos modulos. Si se usa JUnit 4, tener cuidado con conflictos de versiones, ya que JUnit 4.12 incluye Hamcrest 1.3 como dependencia transitiva. La solucion es declarar Hamcrest 2.x explicitamente para sobrescribir la version transitiva.

```xml
<!-- Forzar version 2.x para evitar conflicto con JUnit 4 -->
<dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest</artifactId>
    <version>2.2</version>
    <scope>test</scope>
</dependency>
```

Los matchers se importan mediante `import static org.hamcrest.Matchers.*`, que proporciona acceso a todos los metodos fabrica de matchers estandar. Para proyectos grandes, se recomienda importar solo los matchers necesarios para mantener la legibilidad y evitar colisiones de nombres.
