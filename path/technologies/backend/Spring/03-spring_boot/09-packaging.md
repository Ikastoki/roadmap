# Packaging

Spring Boot soporta múltiples opciones de empaquetado para distribuir aplicaciones, siendo el JAR ejecutable la opción más común. El empaquetado determina cómo se distribuye y ejecuta la aplicación, así como las dependencias del servidor de aplicaciones externo. La elección del tipo de empaquetado depende del entorno de despliegue y los requisitos de la organización.

El JAR ejecutable es la opción por defecto y más utilizada. Spring Boot crea un JAR que contiene todas las dependencias de la aplicación, incluyendo el servidor web embebido. Este JAR se puede ejecutar directamente con `java -jar mi-aplicacion.jar`, sin necesidad de un servidor de aplicaciones externo. La estructura interna del JAR sigue un formato especial que permite cargar las dependencias en tiempo de ejecución.

```xml
<!-- Configuración en pom.xml para JAR ejecutable -->
<packaging>jar</packaging>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

```groovy
// Configuración en build.gradle para JAR ejecutable
plugins {
    id 'org.springframework.boot'
    id 'io.spring.dependency-management'
}

jar {
    archiveBaseName = 'mi-aplicacion'
    archiveVersion = '1.0.0'
}
```

El servidor embebido (Tomcat, Jetty o Undertow) se incluye automáticamente dentro del JAR ejecutable. Cuando se ejecuta el JAR, Spring Boot inicializa el servidor web embebido y despliega la aplicación en él. Esto elimina la necesidad de configurar y administrar un servidor de aplicaciones separado, simplificando significativamente el despliegue.

El WAR desplegable es otra opción que permite desplegar la aplicación en un servidor de aplicaciones externo como WildFly, WebLogic o IBM WebSphere. Para crear un WAR, se debe cambiar el empaquetado a `war` y marcar las dependencias del servidor web como proporcionadas. Esto es útil en entornos corporativos que requieren servidores de aplicaciones específicos.

```xml
<!-- Configuración para WAR desplegable -->
<packaging>war</packaging>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

```java
// Clase de aplicación para WAR
@SpringBootApplication
public class MiAplicacion extends SpringBootServletInitializer {
    
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(MiAplicacion.class);
    }
    
    public static void main(String[] args) {
        SpringApplication.run(MiAplicacion.class, args);
    }
}
```

El Spring Boot Maven Plugin proporciona comandos para empaquetar la aplicación. El comando `mvn package` crea el JAR o WAR ejecutable, mientras que `mvn spring-boot:run` ejecuta la aplicación directamente desde el código fuente sin necesidad de empaquetar primero. El plugin también genera archivos de manifiesto que incluyen la clase principal y las dependencias de classpath.

```bash
# Crear JAR ejecutable
mvn clean package

# Ejecutar aplicación directamente
mvn spring-boot:run

# Ejecutar JAR producido
java -jar target/mi-aplicacion-1.0.0.jar
```

```groovy
// Comandos equivalentes en Gradle
// Crear JAR ejecutable
./gradlew bootJar

// Ejecutar aplicación directamente
./gradlew bootRun

// Ejecutar JAR producido
java -jar build/libs/mi-aplicacion-1.0.0.jar
```

El Spring Boot Gradle Plugin ofrece funcionalidades similares a la versión Maven. El task `bootJar` crea el JAR ejecutable, mientras que `bootRun` ejecuta la aplicación. Gradle permite una personalización más flexible del proceso de empaquetado mediante configuración en el archivo `build.gradle`.

> Al crear un WAR para despliegue en servidor externo, las dependencias del servidor web deben marcarse como `provided` para evitar conflictos con el servidor de aplicaciones. Esto asegura que se use la versión del servidor web del contenedor, no la embebida en la aplicación.

