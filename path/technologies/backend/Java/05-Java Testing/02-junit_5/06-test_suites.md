# Test suites

Las suites de tests permiten agrupar y ejecutar multiples clases de test de forma coordinada. JUnit 5 introduce un modelo de suites basado en anotaciones y en la JUnit Platform, reemplazando los runners y anotaciones especificas de JUnit 4 (@RunWith(Suite.class), @SuiteClasses) por un mecanismo mas flexible y expresivo.

## @Suite

La anotacion @Suite marca una clase como suite de tests. Esta clase actua como un contenedor que no contiene tests propios, sino que declara que otros tests deben ser ejecutados como parte de la suite. Para habilitar el soporte de suites, es necesario incluir la dependencia `junit-platform-suite-api` en el proyecto.

```java
import org.junit.platform.suite.api.Suite;

@Suite
class SuiteCompleta {
    // Clase vacia: solo actua como contenedor
}
```

Ademas de la anotacion, se requiere el motor de descubrimiento de suites, que se habilita con la dependencia `junit-platform-suite-engine`. Sin este motor, las suites no seran reconocidas por el launcher de la Platform.

## @SelectClasses

Especifica explicitamente las clases de test que forman parte de la suite. Es el metodo mas directo y el que ofrece mayor control sobre que tests se incluyen. Es recomendable cuando el numero de clases es manejable y se quiere una seleccion precisa sin depender de convenciones de paquete.

```java
@Suite
@SelectClasses({ UsuarioServiceTest.class, FacturaServiceTest.class, ProductoServiceTest.class })
class SuiteModuloFacturacion {
}
```

## @SelectPackages

Selecciona todos los tests que se encuentran dentro de uno o varios paquetes especificados. Es mas flexible que SelectClasses porque incluye automaticamente cualquier clase de test que se anada al paquete en el futuro. Sin embargo, puede incluir tests no deseados si la estructura de paquetes no esta bien definida.

```java
@Suite
@SelectPackages({ "com.empresa.modulo.usuario", "com.empresa.modulo.facturacion" })
class SuiteModulos {
}
```

La seleccion por paquetes puede combinarse con filtros adicionales para refinar la inclusion.

## @IncludeTags y @ExcludeTags

Filtran los tests incluidos en la suite segun sus anotaciones @Tag. Permiten crear suites semanticas que ejecuten solo tests con determinadas etiquetas (rapidos, lentos, integracion, api) sin necesidad de listar clases individuales. Las expresiones soportan los operadores logicos definidos por la JUnit Platform.

```java
@Suite
@SelectPackages("com.empresa")
@IncludeTags("rapido")
@ExcludeTags("lento")
class SuiteTestsRapidos {
}
```

Tambien es posible combinar inclusion y exclusion con SelectClasses para un control granular:

```java
@Suite
@SelectClasses({ UsuarioServiceTest.class, FacturaServiceTest.class })
@IncludeTags("integracion")
@ExcludeTags("db")
class SuiteIntegracionSinBaseDatos {
}
```

> El uso combinado de @IncludeTags y @ExcludeTags permite implementar perfiles de ejecucion (smoke, regression, full) que se activan en diferentes momentos del pipeline de CI/CD.

## Ejecucion selectiva con ConfigurationParameters

La JUnit Platform permite configurar parametros de ejecucion a traves del fichero `junit-platform.properties`. Estos parametros incluyen filtros globales de tags, paralelismo y configuracion de descubrimiento.

```properties
# junit-platform.properties
junit.jupiter.conditions.deactivate=org.junit.*DisabledCondition
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
```

La combinacion de suites con configuracion externa proporciona un sistema flexible donde las suites definen la estructura logica de los tests y la configuracion determina como se ejecutan (en paralelo, con ciertos filtros, con condiciones especificas de entorno).
