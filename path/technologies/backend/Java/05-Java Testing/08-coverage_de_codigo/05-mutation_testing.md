# Mutacion Testing (PIT)

El mutation testing es una tecnica de evaluacion de la calidad de los tests que va un paso mas alla del code coverage. En lugar de medir que lineas se ejecutan, introduce cambios deliberados (mutaciones) en el codigo de produccion y comprueba si los tests existentes son capaces de detectarlos. Si un test falla tras la mutacion, el mutante muere. Si el test sigue pasando, el mutante sobrevive, lo que indica que hay una debilidad en la suite de tests.

## Mutantes

Un mutante es una version modificada del codigo original aplicando operadores de mutacion. Los operadores mas comunes incluyen:

- Cambiar operadores relacionales: `<` por `<=`, `>` por `>=`
- Negar condiciones: `if (activo)` por `if (!activo)`
- Reemplazar constantes: `return 0` por `return 1`, `return -1`
- Eliminar llamadas a metodos: omitir una invocacion
- Cambiar operadores aritmeticos: `+` por `-`, `*` por `/`
- Modificar valores booleanos: `true` por `false`

```java
// Original
if (edad >= 18) {
    return true;
}
return false;

// Mutante 1: cambiar >= por >
if (edad > 18) { ... }

// Mutante 2: negar condicion
if (!(edad >= 18)) { ... }
```

Si el test que cubre esta logica no distingue entre `edad >= 18` y `edad > 18`, el mutante sobrevive y la prueba no es suficientemente precisa.

## Kill rate

El kill rate es la proporcion de mutantes que los tests eliminan. Un kill rate del 100 % indica que los tests detectan cualquier variacion superficial del codigo. En la practica, valores superiores al 90 % se consideran excelentes. Por debajo del 70 % hay una debilidad clara en la bateria de tests.

La metrica de mutation testing es mucho mas reveladora que el code coverage porque mide la capacidad de detectar errores, no solo de ejecutar codigo. Es posible tener un 100 % de line coverage y un 50 % de kill rate.

## PIT (Pitest)

PIT es la herramienta de mutation testing de referencia para Java. Se integra con Maven y Gradle, es compatible con JUnit 5 y Mockito, y esta optimizada para minimizar el tiempo de ejecucion mediante tecnicas como mutacion selectiva y caching de resultados.

```xml
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
    <configuration>
        <targetClasses>
            <param>com.miempresa.servicio.*</param>
        </targetClasses>
        <targetTests>
            <param>com.miempresa.servicio.*</param>
        </targetTests>
        <mutationThreshold>80</mutationThreshold>
    </configuration>
</plugin>
```

```bash
mvn org.pitest:pitest-maven:mutationCoverage
```

En Gradle, se utiliza el plugin `info.solidsoft.pitest`:

```groovy
plugins {
    id 'info.solidsoft.pitest' version '1.15.0'
}

pitest {
    junit5PluginVersion = '1.2.1'
    targetClasses = ['com.miempresa.servicio.*']
    targetTests = ['com.miempresa.servicio.*']
    mutationThreshold = 80
}
```

PIT genera un reporte HTML que muestra el codigo original con los mutantes que sobrevivieron marcados en rojo. Cada mutante incluye el operador aplicado y el motivo por el que no fue detectado.

## Ventajas sobre coverage

El mutation testing resuelve las limitaciones fundamentales del code coverage:

- Un test que ejecuta codigo pero nunca falla tiene cobertura pero no protege. PIT detecta esa falsa sensacion de seguridad.
- Revela aserciones debiles o ausentes. Si un metodo devuelve un valor y el test solo verifica que no lance excepcion, PIT detectara mutantes que alteran el resultado.
- Incentiva tests mas precisos. Para matar mutantes, los tests deben verificar el comportamiento concreto, no solo la ausencia de errores.

> El mutation testing es computacionalmente costoso. Cada mutante requiere ejecutar la suite completa de tests, lo que puede multiplicar el tiempo de ejecucion por decenas o cientos. En proyectos grandes se recomienda ejecutarlo de forma selectiva o solo en CI nocturno.
