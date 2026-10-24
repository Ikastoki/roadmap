# Spring Native

GraalVM es una maquina virtual poliglota desarrollada por Oracle que incluye un compilador Ahead-Of-Time (AOT) capaz de convertir aplicaciones Java en imagenes nativas independientes. Estas imagenes nativas eliminan la necesidad de una JVM en el momento de ejecucion, reduciendo dramaticamente el tiempo de arranque (de segundos a milisegundos) y el consumo de memoria residente. Spring Native integra el ecosistema Spring con GraalVM, permitiendo construir aplicaciones Spring Boot que se ejecutan como imagenes nativas.

## Compilacion AOT vs JIT

La compilacion Just-In-Time (JIT) tradicional de la JVM compila el bytecode a codigo nativo durante la ejecucion, optimizando las rutas de codigo mas ejecutadas (*hot paths*). Este enfoque es ideal para servidores de larga duracion donde el coste de arranque se amortiza. La compilacion AOT, en cambio, genera el codigo nativo completo antes de la ejecucion, en tiempo de build. Esto elimina la fase de calentamiento del JIT pero requiere que todas las clases, metodos y reflexion sean conocidos estaticamente.

```bash
# Compilacion AOT de una aplicacion Spring
mvn -Pnative native:compile

# Ejecucion de la imagen nativa
./target/mi-aplicacion
```

El resultado es un ejecutable binario sin dependencia de JVM, sin necesidad de claspath y sin resolucion dinamica de clases en tiempo de ejecucion. El arranque se completa en menos de 100ms y el consumo de memoria se reduce significativamente.

## Dependencias

Spring Boot 3 integra soporte nativo de GraalVM de forma nativa a traves del starter `spring-boot-starter-aot` y el plugin Maven `spring-boot-maven-plugin` con perfil `native`. No se requiere la dependencia separada `spring-native` que existia en versiones anteriores.

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <image>
                    <builder>paketobuildpacks/builder-jammy-native:latest</builder>
                </image>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```xml
<profiles>
    <profile>
        <id>native</id>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-aot</artifactId>
            </dependency>
        </dependencies>
    </profile>
</profiles>
```

## Proceso de compilacion

El plugin de GraalVM ejecuta un proceso de varias fases para generar la imagen nativa:

1. **Fase AOT de Spring**: Spring ejecuta la aplicacion en modo AOT, resolviendo todos los beans, configuraciones y rutas de codigo que se usaran en tiempo de ejecucion. Genera archivos de hints y un `BeanFactory` optimizado.
2. **Generacion de puntos de entrada**: GraalVM analiza el bytecode y genera un archivo de configuracion que indica las clases que necesitan soporte de reflexion, recursos y proxies.
3. **Compilacion nativa**: El compilador GraalVM convierte el bytecode optimizado a codigo maquina nativo, eliminando codigo no alcanzable y aplicando optimizaciones estaticas.

```bash
# Ejecutar en modo AOT (requiere GraalVM instalado)
java -agentlib:native-image-agent=config-output-dir=META-INF/native-image \
    -jar target/mi-aplicacion.jar

# Compilar imagen nativa
mvn -Pnative native:compile -DskipTests
```

> La generacion de la imagen nativa puede requerir 5-15 minutos dependiendo del tamano de la aplicacion. El tiempo de arranque resultante se mide en milisegundos, lo que lo ideal para funciones serverless, contenedores de corta vida y escenarios de escalado rapido.

## Ventajas en produccion

Las imagenes nativas de Spring Boot presentan ventajas concretas en entornos productivos:

- **Arranque instantaneo**: menos de 100ms para aplicaciones complejas, ideal para autoescalado en Kubernetes
- **Consumo de memoria reducido**: tipicamente 3-5x menos que una JVM convencional
- **Sin tiempo de calentamiento**: la optimizacion completa esta disponible desde el primer request
- **Distribucion simplificada**: un unico binario ejecutable sin dependencias de runtime

```bash
# Ejecutar la imagen nativa
./target/mi-aplicacion --server.port=8080

# Verificar tiempos de arranque
time ./target/mi-aplicacion &
# Resultado: ~50ms para arrancar completamente
```

El trade-off principal es un tiempo de compilacion significativamente mayor y la necesidad de declarar explicitamente las clases que requieren reflexion y proxies dinamicos. En entornos de desarrollo donde el ciclo rapido es prioritario, la compilacion JIT sigue siendo preferible, reservando las imagenes nativas para despliegues en produccion o entornos de integracion continua.

