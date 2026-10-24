# Estructura de test

La estructura de directorios y la organizacion de las clases de test es una decision arquitectonica que afecta a la mantenibilidad, la ejecucion en CI y la colaboracion en equipo. Los proyectos Java siguen convenciones ampliamente aceptadas que conviene respetar.

## src/test/java vs src/main/java

Maven y Gradle establecen dos arboles de origen diferenciados:

```
src/
  main/
    java/       -> codigo de produccion
    resources/  -> recursos de produccion (properties, XML, YAML)
  test/
    java/       -> codigo de test
    resources/  -> recursos de test (datos de prueba, configuraciones)
```

El codigo en `src/main/java` se empaqueta en el artefacto de produccion. El codigo en `src/test/java` solo existe durante la compilacion y ejecucion de tests. Esta separacion garantiza que las dependencias de test no se filtren al classpath de produccion.

## Paquetes espejo

La convencion mas extendida es que los tests reflejen la estructura de paquetes del codigo de produccion. Una clase `com.miempresa.servicio.UsuarioService` tiene su test en `com.miempresa.servicio.UsuarioServiceTest`.

```
src/main/java/com/miempresa/servicio/
  UsuarioService.java
  FacturaService.java

src/test/java/com/miempresa/servicio/
  UsuarioServiceTest.java
  FacturaServiceTest.java
```

Esta coincidencia de paquetes permite que los tests accedan a miembros con visibilidad de paquete (package-private) sin necesidad de exponerlos publicamente. Es una practica recomendada para mantener el encapsulamiento.

## src/test/resources

El directorio `src/test/resources` contiene archivos auxiliares para los tests: archivos JSON de prueba, schemas, bases de datos H2 en memoria, configuraciones de log, etc. Estos recursos se anaden al classpath de test automaticamente.

```
src/test/resources/
  datos/
    usuario-valido.json
    factura-error.csv
  application-test.yml
  logback-test.xml
```

## Convenciones de organizacion

Dentro de `src/test/java`, los tests se agrupan por funcionalidad o por capa:

- Tests unitarios en el mismo paquete que la clase bajo prueba.
- Tests de integracion en un subpaquete `integration` o con el sufijo `IT`.
- Tests de contratos o componentes en paquetes separados.

```
src/test/java/com/miempresa/
  servicio/
    UsuarioServiceTest.java
    FacturaServiceTest.java
  integration/
    UsuarioServiceIT.java
    FacturaServiceIT.java
  repository/
    UsuarioRepositoryTest.java
```

## @DisplayName

La anotacion `@DisplayName` de JUnit 5 permite describir el test con texto legible, incluyendo espacios y caracteres especiales. Es util para generar reportes comprensibles.

```java
@DisplayName("Servicio de usuarios")
class UsuarioServiceTest {

    @Test
    @DisplayName("Debe lanzar excepcion cuando el email ya existe")
    void lanzaExcepcionCuandoEmailYaExiste() {
        // ...
    }
}
```

Los `@DisplayName` deben escribirse en el mismo idioma que el equipo usa para la documentacion. Si el equipo es hispanohablante, los nombres descriptivos en espanol mejoran la comunicacion.

> La estructura de tests debe reflejar la arquitectura del proyecto. Si el codigo de produccion cambia su organizacion, los tests deben reorganizarse en consecuencia. Mantener la correspondencia evita confusiones y reduce el costo de incorporar nuevos miembros al equipo.
