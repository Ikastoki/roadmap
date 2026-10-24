# JaCoCo

JaCoCo (Java Code Coverage) es la herramienta de cobertura de codigo mas extendida en el ecosistema Java. Opera instrumentando el bytecode en tiempo de ejecucion, lo que significa que no requiere modificar el codigo fuente ni recompilar. Soporta line coverage, branch coverage y method coverage, y genera reportes en multiples formatos: HTML, XML, CSV y una estructura binaria que puede consumir SonarQube.

## Plugin de Maven

La integracion con Maven se realiza mediante el plugin `jacoco-maven-plugin`. La configuracion tipica prepara el agente de JaCoCo antes de ejecutar los tests y genera el reporte al finalizar.

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

El goal `prepare-agent` anade el argumento `-javaagent` al JVM que ejecuta los tests. El goal `report` genera el informe HTML en `target/site/jacoco/`.

## Plugin de Gradle

En Gradle se aplica el plugin oficial `jacoco`. Basta con declararlo en `build.gradle` para obtener las tareas `jacocoTestReport` y `jacocoTestCoverageVerification`.

```groovy
plugins {
    id 'java'
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.12"
}

test {
    useJUnitPlatform()
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    dependsOn test
    reports {
        html.required = true
        xml.required = true
    }
}
```

## Reportes HTML

El reporte HTML de JaCoCo colorea el codigo fuente: verde para lineas cubiertas, rojo para no cubiertas, amarillo para cubiertas parcialmente (branch coverage incompleto). Cada clase muestra un resumen con el porcentaje de instrucciones, ramas, metodos y clases. La navegacion por paquetes permite identificar rapidamente los modulos con menor cobertura.

## Exclusiones

Es frecuente excluir del報告 codigo generado, DTOs, configuraciones o clases de infraestructura que no requieren tests. JaCoCo permite exclusiones a nivel de clase y metodo mediante expresiones regulares.

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>**/dto/**</exclude>
            <exclude>**/Application.class</exclude>
        </excludes>
    </configuration>
</plugin>
```

Tambien se puede marcar un metodo con la anotacion `@Generated` para que JaCoCo lo ignore, util en codigo generado por Lombok.

## Quality gates

JaCoCo incluye una tarea de verificacion que puede fallar el build si la cobertura no alcanza un minimo. Se configura mediante `check` en Maven o la tarea `jacocoTestCoverageVerification` en Gradle.

```xml
<execution>
    <id>check</id>
    <goals><goal>check</goal></goals>
    <configuration>
        <rules>
            <rule>
                <element>PACKAGE</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.80</minimum>
                    </limit>
                </limits>
            </rule>
        </rules>
    </configuration>
</execution>
```

Cuando se incumple la regla, el build falla con un mensaje que indica el porcentaje actual y el minimo exigido. Esto convierte el coverage en un control automatico dentro del pipeline de CI/CD.

> JaCoCo no puede medir cobertura de codigo que nunca se carga en el classpath. Si una clase no se instancia, no aparecera en el reporte, lo que puede dar una falsa sensacion de completitud.
