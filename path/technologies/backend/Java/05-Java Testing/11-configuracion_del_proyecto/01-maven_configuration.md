# Maven configuration

Maven es el sistema de build mas utilizado en el ecosistema Java. Su modelo basado en plugins y ciclos de vida permite integrar todas las herramientas de testing de forma declarativa en el `pom.xml`.

## maven-surefire-plugin

El plugin Surefire es el responsable de ejecutar los tests unitarios durante la fase `test`. Detecta automaticamente las clases de test siguiendo convenciones de nomenclatura (`*Test.java`, `Test*.java`, `*Tests.java`) y las ejecuta con JUnit 5, JUnit 4 o TestNG segun las dependencias presentes.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>
        <excludes>
            <exclude>**/*IT.java</exclude>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
        <forkCount>2</forkCount>
        <reuseForks>true</reuseForks>
    </configuration>
</plugin>
```

El parametro `forkCount` controla cuantos procesos JVM se lanzan en paralelo para ejecutar tests. `reuseForks` reutiliza las JVM entre clases de test para reducir el overhead.

## maven-failsafe-plugin

Failsafe esta disenado para tests de integracion. Se ejecuta en la fase `verify`, despues del empaquetado, y no falla el build si los tests unitarios fallan (eso es responsabilidad de Surefire). La convencion tipica es nombrar los tests de integracion con el sufijo `*IT.java` o `*IntegrationTest.java`.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.5</version>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <includes>
            <include>**/*IT.java</include>
        </includes>
    </configuration>
</plugin>
```

## JaCoCo plugin

El plugin de JaCoCo se integra con el ciclo de vida de Maven para instrumentar el bytecode antes de los tests y generar el reporte de cobertura en la fase `verify`.

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
    </executions>
</plugin>
```

## Pitest (PIT)

PIT se integra mediante el plugin `pitest-maven`. Requiere el conector de JUnit 5 para detectar los tests escritos con este framework.

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

## pom.xml completo

Un `pom.xml` con testing completo incluye las dependencias de JUnit 5, Mockito y AssertJ junto con los plugins de Surefire, Failsafe y JaCoCo. La configuracion se mantiene en un unico archivo que cualquier desarrollador puede reproducir localmente y en CI sin variaciones.

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.25.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

La separacion entre Surefire (unitarios) y Failsafe (integracion) permite ejecutar los tests rapidos en cada commit y reservar los lentos para fases posteriores del pipeline. Combinado con JaCoCo y PIT, se obtiene un sistema de calidad automatizado que verifica correccion, cobertura y efectividad de los tests.
