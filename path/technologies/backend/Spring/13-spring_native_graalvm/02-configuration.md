# Configuration

GraalVM genera imagenes nativas mediante analisis estatico del bytecode, lo que significa que no puede resolver en tiempo de ejecucion clases que se cargan por reflexion, proxies generados dinamicamente o recursos que se cargan desde el classpath de forma condicional. Spring ofrece un conjunto de mecanismos para declarar estos elementos de forma explicita, permitiendo que el compilador nativo incluya la informacion necesaria en la imagen final.

## RegisterReflectionForBinding

La anotacion `@RegisterReflectionForBinding` indica al compilador GraalVM que una clase necesita soporte completo de reflexion en la imagen nativa. Esto incluye acceso a campos, constructores, metodos y anotaciones en tiempo de ejecucion.

```java
@RegisterReflectionForBinding({
    Usuario.class,
    Producto.class,
    Pedido.class
})
@Configuration
public class ReflectionConfig {
}
```

Para clases que se utilizan en serializacion JSON o binaria, es necesario registrar tanto el tipo como sus campos accesibles:

```java
@Configuration
@ImportRuntimeHints(AppRuntimeHints.class)
public class ReflectionConfig {
}
```

> Sin los hints de reflexion, GraalVM eliminará las clases no alcanzables estaticamente del binario final, provocando `ClassNotFoundException` o `InaccessibleObjectException` en tiempo de ejecucion.

## Runtime hints

`RuntimeHints` es la API de bajo nivel que permite registrar hints para reflexion, recursos, proxies y serializacion JNI. Spring Boot ejecuta automaticamente un proceso de introspeccion en tiempo de AOT para detectar la mayoria de los casos comunes, pero para escenarios personalizados se pueden registrar hints manuales.

```java
@Configuration
@ImportRuntimeHints(AppRuntimeHints.class)
public class RuntimeHintsConfig {
}

public class AppRuntimeHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Reflexion: registrar clase y campos
        hints.reflection()
            .registerType(Usuario.class, MemberCategory.values())
            .registerType(Producto.class, builder ->
                builder.withField("id")
                       .withField("nombre")
                       .withField("precio"));

        // Recursos: registrar archivos que se cargan desde classpath
        hints.resources()
            .registerPattern("templates/*")
            .registerPattern("static/**")
            .registerPattern("META-INF/resources/**");

        // Proxies: registrar interfaces que necesitan proxies dinamicos
        hints.proxies()
            .registerJdkProxy(UsuarioService.class);
    }
}
```

Los `MemberCategory` controlan el nivel de acceso a la reflexion:

- `DECLARED_FIELDS`: campos declarados en la clase
- `PUBLIC_FIELDS`: campos publicos heredados
- `DECLARED_METHODS`: metodos declarados en la clase
- `PUBLIC_METHODS`: metodos publicos heredados
- `INTROSPECT_PUBLIC_CONSTRUCTORS`: introspeccion de constructores publicos
- `INTROSPECT_DECLARED_CONSTRUCTORS`: introspeccion de constructores declarados
- `INVOKE_PUBLIC_CONSTRUCTORS`: invocacion de constructores publicos
- `INVOKE_DECLARED_CONSTRUCTORS`: invocacion de constructores declarados
- `INVOKE_PUBLIC_METHODS`: invocacion de metodos publicos
- `INVOKE_DECLARED_METHODS`: invocacion de metodos declarados

## Resource hints

Los recursos que se cargan desde el classpath (como archivos de configuracion, plantillas o propiedades) deben registrarse explicitamente para que GraalVM los incluya en la imagen nativa. Spring Boot registra automaticamente los recursos de los modulos de Spring y los archivos bajo `META-INF/resources/`, pero los recursos personalizados requieren hints manuales.

```java
public class RecursosHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.resources()
            .registerPattern("reports/*.jasper")
            .registerPattern("messages/*.properties")
            .registerPattern("schemas/*.xsd");

        hints.resources()
            .registerPattern("META-INF/spring.factories")
            .registerPattern("META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports");
    }
}
```

## Proxy hints

Los proxies dinamicos se utilizan extensivamente en Spring para AOP, transacciones y seguridad. GraalVM requiere que cada interfaz que se use como proxy se registre explicitamente:

```java
public class ProxyHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.proxies()
            .registerJdkProxy(
                UsuarioService.class,
                Auditable.class
            );
    }
}
```

Los proxies CGLIB (utilizados por `@Configuration`, `@Bean` y `@Scope`) se registran automaticamente por Spring AOT. Solo es necesario registrar proxies JDK manualmente para interfaces que se usen con `java.lang.reflect.Proxy`.

## Validacion de hints

El plugin de GraalVM incluye herramientas para validar que todos los hints estan correctamente configurados antes de generar la imagen nativa:

```bash
# Ejecutar en modo AOT con trazas detalladas
mvn -Pnative native:compile -Dagent

# Analizar los hints generados
cat target/native-image/reflect-config.json
cat target/native-image/resource-config.json
cat target/native-image/proxy-config.json
```

Estos archivos JSON contienen la representacion final de los hints que GraalVM utilizara durante la compilacion nativa. Inspeccionarlos permite verificar que todas las clases, recursos y proxies necesarios estan incluidos.

Spring Boot 3 ejecuta la mayoria de los hints de forma automatica mediante `RuntimeHintsDetector`, que analiza el contexto de la aplicacion y genera los hints necesarios. Solo es necesario registrar hints manuales para casos no estandar como serializacion personalizada, clases cargadas por nombre dinamicamente o recursos cargados con rutas calculadas en tiempo de ejecucion.

