# Application Properties

Spring Boot permite externalizar la configuración de la aplicación mediante archivos de propiedades, proporcionando una forma flexible de gestionar configuraciones diferentes sin modificar el código fuente. Los archivos de configuración se cargan automáticamente al iniciar la aplicación y se aplican a las beans que los necesitan.

`application.properties` es el archivo de configuración principal en formato key-value estándar de Java. Cada línea contiene una propiedad en el formato `clave=valor`. Este formato es simple pero puede volverse verboso para configuraciones jerárquicas complejas.

`application.yml` es una alternativa al formato de propiedades que utiliza YAML (YAML Ain't Markup Language). YAML proporciona una estructura más legible y jerárquica, lo que facilita la organización de configuraciones complejas. Ambos formatos son equivalentes y se pueden usar indistintamente, aunque no se recomienda mezclarlos en el mismo proyecto.

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mibd
spring.datasource.username=root
spring.datasource.password=secret
```

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mibd
    username: root
    password: secret
```

El orden de las fuentes de propiedades (property sources) es crítico para entender cómo se resuelven las configuraciones. Spring Boot carga las propiedades en un orden específico donde las propiedades posteriores sobreescriben a las anteriores. El orden típico es: propiedades de command-line, propiedades de sistema (`-D`), variables de entorno, archivos `application-{profile}.properties` específicos del perfil y finalmente `application.properties` principal.

La configuración multi-perfil permite definir configuraciones diferentes para cada entorno (desarrollo, testing, producción). Spring Boot detecta automáticamente los archivos `application-{profile}.properties` y los carga según el perfil activo. El perfil se puede activar mediante la propiedad `spring.profiles.active` o la variable de entorno `SPRING_PROFILES_ACTIVE`.

```properties
# application-dev.properties
server.port=8080
spring.datasource.url=jdbc:h2:mem:testdb

# application-prod.properties
server.port=443
spring.datasource.url=jdbc:mysql://prod-server:3306/proddb
spring.datasource.username=prod_user
```

`@ConfigurationProperties` permite strongly-typed binding de propiedades a objetos Java. Esta anotación se puede usar en clases de configuración para agrupar propiedades relacionadas y proporcionar validación automática. El atributo `prefix` define el prefijo de las propiedades que se mapearán a los campos de la clase.

```java
@ConfigurationProperties(prefix = "mi.app.datasource")
public class MiDataSourceProperties {
    private String url;
    private String username;
    private String password;
    private int poolSize;
    
    // Getters y setters
}
```

`@EnableConfigurationProperties` se usa en clases de configuración para habilitar el soporte de `@ConfigurationProperties` para una clase específica. Esta anotación registra la clase como una bean de configuración y permite la inyección de dependencias en ella.

El relaxed binding es una característica que permite que las propiedades se definan en múltiples formatos. Por ejemplo, una propiedad `mi.app.datasource.url` se puede definir como `mi.app.datasource.url`, `MI_APP_DATASOURCE_URL`, `miApp.datasource.url` o `mi-app-datasource-url`. Spring Boot normaliza automáticamente estos formatos, lo que facilita la configuración en diferentes entornos.

La externalización de configuración es fundamental para aplicaciones cloud-native. Spring Boot soporta múltiples fuentes de configuración: argumentos de línea de comandos, variables de entorno, JNDI, archivos de propiedades específicos del perfil y configuración centralizada con Spring Cloud Config. Esta flexibilidad permite adaptar la misma aplicación a diferentes entornos sin modificar el código.

Las variables de entorno son especialmente útiles en contenedores Docker y orquestadores como Kubernetes. Spring Boot convierte automáticamente las propiedades de formato kebab-case a variables de entorno en SCREAMING_SNAKE_CASE. Por ejemplo, `spring.datasource.url` se puede configurar mediante la variable de entorno `SPRING_DATASOURCE_URL`.

Los placeholders `${...}` permiten referenciar otras propiedades dentro de los archivos de configuración. Esto evita la duplicación de valores y facilita la gestión de configuraciones que dependen de otras propiedades. Si la propiedad referenciada no existe, se puede proporcionar un valor por defecto usando `:`.

```properties
app.name=Mi Aplicacion
app.description=${app.name} es una app de ejemplo
app.version=${APP_VERSION:1.0.0}
```

Las expresiones SpEL `#{...}` (Spring Expression Language) proporcionan funcionalidad más avanzada en las propiedades. A diferencia de los placeholders que solo resuelven propiedades, SpEL permite expresiones complejas como cálculos, acceso a métodos y operaciones condicionales. Sin embargo, SpEL no se puede usar directamente en `application.properties` sino en anotaciones como `@Value`.

> Es importante no confundir placeholders `${...}` con SpEL `#{...}`. Los placeholders son una función de resolución de propiedades de Spring, mientras que SpEL es un lenguaje de expresión completo con capacidades mucho más amplias.

