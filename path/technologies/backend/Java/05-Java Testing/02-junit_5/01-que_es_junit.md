# Que es JUnit

JUnit es el framework de pruebas unitarias mas extendido en el ecosistema Java. Su primera version, inspirada en SUnit de Smalltalk, sento las bases del testing en Java con anotaciones, aserciones y runners. Con la version 5, JUnit representa una reescritura completa que moderniza la arquitectura, amplia el modelo de extensiones y se adapta a las necesidades del desarrollo Java contemporaneo.

## JUnit 4 vs JUnit 5

JUnit 4 introdujo el modelo de anotaciones (@Test, @Before, @After, @BeforeClass, @AfterClass) que se convirtio en el estandar de la industria durante mas de una decada. Sin embargo, su arquitectura presentaba limitaciones crecientes: el modelo de extensiones (via @RunWith y Rules) era restrictivo, no permitia la ejecucion paralela nativa y dependia del mecanismo de descubrimiento de tests basado en convenciones de nomenclatura. JUnit 5 aborda todas estas limitaciones con un diseno modular que separa claramente las responsabilidades de descubrimiento, ejecucion y reporte.

| Aspecto | JUnit 4 | JUnit 5 |
|---------|---------|---------|
| Extensiones | @RunWith, @Rule, @ClassRule | @ExtendWith, interfaces de extension |
| Descubrimiento | Convencion de nombres | ServiceLoader + anotacion @Testable |
| Version Java minima | Java 5 | Java 8 |
| Ejecucion paralela | No nativa | Nativa |
| Tests parametrizados | Limitado | Robusto y flexible |

## Arquitectura de JUnit 5

JUnit 5 se compone de tres modulos principales que forman una arquitectura en capas claramente separadas.

### JUnit Platform

Es la base sobre la que se ejecutan los frameworks de testing en la JVM. Proporciona la API de lanzamiento (Launcher API) que permite a IDEs, herramientas de build (Maven, Gradle) y consolas descubrir y ejecutar tests. La Platform tambien define el TestEngine SPI (Service Provider Interface) que permite integrar cualquier framework de testing (JUnit 4, JUnit 5, TestNG, Spock) bajo un mismo mecanismo de ejecucion. La consola y el mecanismo de descubrimiento via ServiceLoader forman parte de este modulo.

### JUnit Jupiter

Jupiter es el corazon de JUnit 5: contiene el nuevo modelo de programacion (anotaciones, aserciones, asunciones) y la implementacion del TestEngine que sabe como descubrir y ejecutar tests escritos con ese modelo. Cuando un desarrollador escribe:

```java
@Test
void miPrimerTest() {
    assertTrue(true);
}
```

Jupiter es el modulo que interpreta la anotacion @Test, recolecta los metodos de prueba, maneja el ciclo de vida y ejecuta las aserciones. Es, en esencia, el reemplazo directo del JUnit 4 clasico, pero con una API mas rica y extensible.

### JUnit Vintage

Vintage proporciona un TestEngine que ejecuta tests escritos con JUnit 4 (y JUnit 3) sobre la JUnit Platform. Esto permite mantener suites de tests existentes mientras se migra gradualmente a JUnit 5. El modulo Vintage actua como un puente de compatibilidad: los tests escritos con @Test de JUnit 4, @Before, @After, @RunWith y Rules se ejecutan sin modificaciones sobre la Platform.

```
+-------------------------------------------+
|              IDEs / Build Tools            |
+-------------------------------------------+
|            JUnit Platform Launcher         |
+-------------------------------------------+
|  Jupiter Engine  |  Vintage Engine  | ... |
|  (JUnit 5 tests) |  (JUnit 4 tests) |     |
+-------------------------------------------+
```

## Dependencias Maven y Gradle

Para comenzar a usar JUnit 5 en un proyecto, se requieren las siguientes dependencias.

**Maven:**
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

El artefacto `junit-jupiter` es un aggregator que incluye `junit-jupiter-api` (API de programacion), `junit-jupiter-engine` (motor de ejecucion) y `junit-jupiter-params` (soporte para tests parametrizados). Para la Platform se puede incluir `junit-platform-suite` si se necesitan suites de tests.

**Gradle:**
```groovy
test {
    useJUnitPlatform()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.11.0'
}
```

La llamada a `useJUnitPlatform()` en la tarea `test` le indica a Gradle que utilice el Launcher de JUnit Platform en lugar del mecanismo clasico basado en anotaciones. Sin esta configuracion, Gradle no descubrira los tests de JUnit 5.

> Aunque JUnit 5 es compatible con Java 8, sus capacidades brillan con Java 11 o superior gracias al soporte de modulos, mejoras en lambdas y la API de Stream que permiten tecnicas avanzadas como DynamicTest y tests parametrizados.
