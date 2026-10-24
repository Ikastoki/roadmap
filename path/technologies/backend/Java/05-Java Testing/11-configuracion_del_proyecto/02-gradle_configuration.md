# Gradle configuration

Gradle ofrece un modelo de configuracion mas conciso que Maven para el testing. Los plugins se aplican mediante el bloque `plugins {}` y las tareas se configuran con DSL de Groovy o Kotlin.

## test task

La tarea `test` de Gradle ejecuta los tests unitarios. Por defecto busca clases que coincidan con los patrones `*Test.java` y `Test*.java` en `src/test/java`.

```groovy
test {
    useJUnitPlatform()
    maxParallelForks = 2
    forkEvery = 50
    testLogging {
        events "passed", "skipped", "failed"
        exceptionFormat "full"
        showStandardStreams = true
    }
}
```

- `useJUnitPlatform()` activa el motor de JUnit 5.
- `maxParallelForks` controla la ejecucion paralela de tests.
- `forkEvery` define cuantas clases de test ejecutar por proceso JVM antes de reiniciarlo.
- `testLogging` configura la salida en consola.

## useJUnitPlatform

Sin esta llamada, Gradle usa JUnit 4 por defecto. Es necesaria para que funcione JUnit 5, jqwik y cualquier otra libreria basada en el TestEngine de JUnit 5.

```groovy
test {
    useJUnitPlatform()
}
```

## testImplementation

Las dependencias de testing se declaran con la configuracion `testImplementation`. Solo estan disponibles en el classpath de compilacion y ejecucion de tests.

```groovy
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testImplementation 'org.mockito:mockito-junit-jupiter:5.11.0'
    testImplementation 'org.assertj:assertj-core:3.25.3'
    testImplementation 'net.jqwik:jqwik:1.8.4'
}
```

## JaCoCo plugin

El plugin `jacoco` anade las tareas `jacocoTestReport` y `jacocoTestCoverageVerification`.

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

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }
        }
    }
}

check.dependsOn jacocoTestCoverageVerification
```

La linea `check.dependsOn jacocoTestCoverageVerification` asegura que el build falle si no se alcanza la cobertura minima.

## Pitest

El plugin `info.solidsoft.pitest` integra PIT en Gradle.

```groovy
plugins {
    id 'info.solidsoft.pitest' version '1.15.0'
}

pitest {
    junit5PluginVersion = '1.2.1'
    targetClasses = ['com.miempresa.servicio.*']
    targetTests = ['com.miempresa.servicio.*']
    mutationThreshold = 80
    threads = 4
    outputFormats = ['HTML', 'XML']
    timestampedReports = false
}
```

## Build completo en Gradle

El archivo `build.gradle` integra todas las herramientas en un unico punto de configuracion:

```groovy
plugins {
    id 'java'
    id 'jacoco'
    id 'info.solidsoft.pitest' version '1.15.0'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testImplementation 'org.mockito:mockito-junit-jupiter:5.11.0'
    testImplementation 'org.assertj:assertj-core:3.25.3'
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

La ventaja de Gradle es que las tareas son explicitas y se pueden componer. Si un proyecto necesita una configuracion especifica para tests de integracion, se define una tarea `integrationTest` separada con su propio juego de dependencias y reglas.
