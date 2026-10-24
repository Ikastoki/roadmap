# Limitaciones

La compilacion de imagenes nativas con GraalVM impone restricciones significativas sobre el tipo de codigo que puede ejecutarse en tiempo de ejecucion. Estas limitaciones derivan del analisis estatico que GraalVM realiza durante la compilacion: el compilador necesita conocer de forma anticipada todas las clases, metodos y recursos que se usaran, lo que elimina la posibilidad de resolver dinamicamente elementos que solo estan disponibles en tiempo de ejecucion.

## Proxies dinamicos

La generacion de proxies dinamicos es una de las areas mas afectadas por las restricciones de GraalVM. En una JVM convencional, `java.lang.reflect.Proxy` puede generar implementaciones de interfaces en tiempo de ejecucion sin declaracion previa. En una imagen nativa, cada interfaz que se use como objetivo de un proxy debe registrarse explicitamente en la configuracion de proxies.

```java
// Esto falla en imagen nativa si UsuarioService no esta registrado como proxy
Proxy.newProxyInstance(
    classLoader,
    new Class[]{UsuarioService.class},
    (proxy, method, args) -> method.invoke(target, args)
);

// Solucion: registrar el proxy en RuntimeHints
hints.proxies().registerJdkProxy(UsuarioService.class);
```

Spring AOP, las transacciones declarativas (`@Transactional`) y la seguridad basada en metodos (`@PreAuthorize`) dependen fuertemente de proxies. Spring Boot 3 detecta automaticamente la mayoria de estos casos durante la fase AOT, pero las interfaz personalizadas que se usen con `@Aspect` o `@Around` requieren registro manual.

## Reflexion

La reflexion en Java permite inspeccionar y modificar clases, campos, metodos y constructores en tiempo de ejecucion. GraalVM elimina la informacion de metadatos de reflexion de la imagen nativa para reducir el tamano, por lo que cada clase que se acceda via reflexion debe registrarse explicitamente.

```java
// Esto falla si Usuario.class no esta registrado para reflexion
Field campo = Usuario.class.getDeclaredField("nombre");
campo.setAccessible(true);
String valor = (String) campo.get(usuario);

// Solucion: registrar la clase y sus campos
hints.reflection()
    .registerType(Usuario.class, MemberCategory.values());
```

Las bibliotecas que dependen extensivamente de la reflexion, como Jackson (serializacion JSON), Hibernate (mapeo ORM) y很多 frameworks de validacion, requieren configuracion adicional para funcionar en imagenes nativas. Spring Boot configura automaticamente las integraciones estandar, pero las bibliotecas de terceros no siempre incluyen hints nativos.

> Jackson es un caso particularmente complejo porque analiza las clases POJO por reflexion para descubrir campos y tipos de datos. Sin los hints adecuados, la serializacion/deserializacion JSON fallara con `MissingPropertyDefinitionException`.

## Escaneo de classpath

El escaneo de classpath es un patron comun en Spring para descubrir beans, configuraciones y componentes automaticamente. En una imagen nativa, el classpath no existe como concepto en tiempo de ejecucion: todas las clases ya estan compiladas en el binario. Esto significa que el escaneo dinamico basado en `ClasspathScanning` no funciona directamente.

```java
// Esto falla en imagen nativa porque no hay classpath que escanear
@ComponentScan(basePackages = "com.miapp")

// Spring Boot resuelve esto en fase AOT, pero la configuracion
// debe ser conocida estaticamente
```

Spring AOT resuelve este problema ejecutando el escaneo de componentes durante la compilacion y generando un `BeanFactory` pre-resuelto. Sin embargo, las clases que se cargan por nombre en tiempo de ejecucion (usando `Class.forName()` o `classpath*:`) no seran detectadas y deben registrarse manualmente.

## Tiempo de compilacion

La generacion de una imagen nativa es un proceso costoso que puede durar varios minutos para una aplicacion compleja. Esto impacta directamente en los tiempos de build en pipelines de integracion continua.

```bash
# Una aplicacion Spring Boot tipica puede tardar 3-10 minutos
mvn -Pnative native:compile

# Compilar solo los hints AOT (mas rapido)
mvn -Pnative spring-boot:process-aot
```

El tiempo de compilacion se incrementa con el numero de clases, dependencias y hints configurados. Para acelerar el proceso en CI/CD, se recomienda:

- Utilizar caché de Maven/Gradle para no recompilar dependencias
- Ejecutar la compilacion nativa solo en branches de produccion
- Mantener una imagen base pre-compilada para actualizaciones incrementales

## Consideraciones de testing

Testing en imagenes nativas presenta desafios adicionales. Los tests unitarios tradicionales se ejecutan sobre la JVM y no validan los hints de GraalVM. Se requieren tests de integracion especificos que ejecuten la imagen nativa compilada.

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>build-native</id>
            <goals>
                <goal>compile-no-fork</goal>
            </goals>
            <phase>integration-test</phase>
        </execution>
    </executions>
</plugin>
```

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@NativeImageTest
class AplicacionIntegrationTest {

    @LocalServerPort
    private int puerto;

    @Test
    void debeArrancarCorrectamente() {
        given()
            .port(puerto)
            .when()
            .get("/api/health")
            .then()
            .statusCode(200);
    }
}
```

Las imagenes nativas no permiten agentes de bytecode en tiempo de ejecucion, lo que imposibilita el uso de herramientas como JaCoCo para cobertura de codigo o arquitectura para analisis de dependencias. La cobertura de codigo debe medirse en tests JVM, no en tests nativos.

