# Dependencias tipicas

Un proyecto Java moderno con testing requiere un conjunto de dependencias bien definidas. Cada libreria cubre un aspecto distinto: el motor de testing, el mocking, las aserciones y las herramientas de cobertura.

## JUnit 5

JUnit 5 esta compuesto por tres modulos principales:

- **junit-jupiter-api**: anotaciones y API para escribir tests (`@Test`, `@BeforeEach`, `@DisplayName`, etc.).
- **junit-jupiter-engine**: implementacion del motor que ejecuta los tests en la plataforma JUnit.
- **junit-jupiter-params**: soporte para tests parametrizados (`@ParameterizedTest`, `@ValueSource`, `@CsvSource`).

La dependencia `junit-jupiter` es un agregador que incluye los tres:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

Para proyectos que necesitan migrar tests de JUnit 4, existe `junit-vintage-engine`, que permite ejecutar tests escritos con JUnit 4 en la plataforma JUnit 5.

```xml
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

## Mockito

Mockito es la libreria de mocking mas usada en Java. La dependencia principal es `mockito-core`, pero para integracion con JUnit 5 se recomienda `mockito-junit-jupiter`, que proporciona `@ExtendWith(MockitoExtension.class)`.

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

Esta dependencia incluye automaticamente `mockito-core` y permite:

- `@Mock` para crear mocks.
- `@InjectMocks` para inyeccion automatica.
- `@Captor` para capturar argumentos.
- `MockitoExtension` que gestiona el ciclo de vida de los mocks.

## AssertJ

AssertJ ofrece aserciones fluidas (fluent API) que mejoran la legibilidad y el mensaje de error. Su version mas reciente es `assertj-core`.

```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.3</version>
    <scope>test</scope>
</dependency>
```

AssertJ proporciona metodos como `assertThat(objeto).isEqualTo().hasSize().containsExactly().extracting().filteredOn()`. Los mensajes de error incluyen informacion contextual: valores esperados, reales y diferencias.

## Hamcrest

Hamcrest fue la libreria de matchers predominante antes de AssertJ. Aunque AssertJ la ha superado en popularidad, Hamcrest sigue usandose en proyectos legacy y en combinacion con Mockito.

```xml
<dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest</artifactId>
    <version>2.2</version>
    <scope>test</scope>
</dependency>
```

Hamcrest ofrece matchers como `is()`, `hasItem()`, `containsInAnyOrder()`, `closeTo()`, `isEmptyString()`.

## JaCoCo

Para cobertura de codigo, el agente JaCoCo se integra como plugin, pero tambien puede incluirse como dependencia si se necesita acceso programatico a sus APIs.

```xml
<dependency>
    <groupId>org.jacoco</groupId>
    <artifactId>org.jacoco.core</artifactId>
    <version>0.8.12</version>
    <scope>test</scope>
</dependency>
```

En la mayoria de los proyectos es suficiente con el plugin de Maven o Gradle, sin dependencia explicita.

## Pitest

PIT requiere el plugin de build y el conector para JUnit 5:

```xml
<!-- Solo plugin, sin dependencia en classpath de test -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
</plugin>
```

## Resumen de dependencias

| Proposito            | Dependencia                         | Scope  |
|----------------------|-------------------------------------|--------|
| Motor de tests       | junit-jupiter (5.10.2)              | test   |
| Mocking              | mockito-junit-jupiter (5.11.0)      | test   |
| Aserciones fluidas   | assertj-core (3.25.3)               | test   |
| Matchers clasicos    | hamcrest (2.2)                      | test   |
| Property-based test  | jqwik (1.8.4)                       | test   |
| Benchmarks           | jmh-core (1.37)                     | test   |
| Cobertura            | jacoco-maven-plugin (0.8.12)        | plugin |
| Mutation testing     | pitest-maven (1.15.0)               | plugin |

Mantener las versiones actualizadas y consistentes evita conflictos de dependencias y asegura compatibilidad con la version de Java del proyecto.
