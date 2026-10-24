# Container Support

Spring Boot genera JARs ejecutables que incluyen todas las dependencias empaquetadas dentro del archivo, lo que facilita su despliegue en contenedores Docker. La creacion de un Dockerfile optimizado para Spring Boot requiere considerar el tamano de la imagen, los tiempos de build, la seguridad y la eficiencia en el uso de capas de cache.

## Dockerfile basico

Un Dockerfile para una aplicacion Spring Boot sigue un patron estandar que aprovecha las capas de Docker para acelerar builds incrementales.

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN addgroup -S spring && adduser -S spring -G spring

USER spring:spring

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

## Multi-stage builds

Las compilaciones multinivel separan la fase de build (compilacion, tests, empaquetado) de la fase de ejecucion (runtime), reduciendo significativamente el tamano de la imagen final.

```dockerfile
# Fase 1: Build
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# Copiar archivos de Maven/Gradle para aprovechar cache
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B

# Copiar codigo fuente y compilar
COPY src/ src/
RUN ./mvnw package -DskipTests -B

# Fase 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN addgroup -S spring && adduser -S spring -G spring

USER spring:spring

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

> El `HEALTHCHECK` de Docker permite a los orquestadores como Docker Swarm o Kubernetes determinar si el contenedor esta saludable y reiniciarlo automaticamente si deja de responder.

## Layered JAR

Spring Boot permite generar JARs con capas separadas que se corresponden con las partes de la aplicacion que cambian con menor frecuencia. Esto optimiza el cache de Docker porque las capas que no cambian (dependencias) se reutilizan entre builds.

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests -B

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Copiar capas en orden de menor a mayor frecuencia de cambio
COPY --from=build /app/target/layers/dependencies/ ./
COPY --from=build /app/target/layers/spring-boot-loader/ ./
COPY --from=build /app/target/layers/snapshot-dependencies/ ./
COPY --from=build /app/target/layers/application/ ./

EXPOSE 8080

ENTRYPOINT ["java", \
    "org.springframework.boot.loader.launch.JarLauncher", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0"]
```

Las capas generadas son: `dependencies` (librerias externas, cambian raramente), `spring-boot-loader` (loader de Spring Boot, casi nunca cambia), `snapshot-dependencies` (dependencias SNAPSHOT, cambian moderadamente) y `application` (codigo de la aplicacion, cambia en cada build).

## Container-optimized packaging

Spring Boot 3 integra soporte nativo para despliegue en contenedores con optimizaciones especificas:

```yaml
# application.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  threads:
    virtual:
      enabled: true
```

```bash
# Empaquetado optimizado para contenedor
mvn spring-boot:build-image -DskipTests

# Esto genera una imagen OCI directamente sin Dockerfile
# usando Cloud Native Buildpacks
```

El comando `spring-boot:build-image` utiliza Buildpacks para generar imagenes OCI optimizadas automaticamente, con multi-stage builds, capas separadas y JRE minimizado. Es una alternativa a Dockerfile que elimina la necesidad de mantener archivos de configuracion de Docker manualmente.

