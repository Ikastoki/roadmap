# Auto-configuration

La auto-configuración es una de las características más poderosas de Spring Boot, permitiendo que el framework configure automáticamente las beans de Spring basándose en las dependencias presentes en el classpath. Este mecanismo analiza las bibliotecas disponibles y configura las beans que considera necesarias, eliminando la configuración manual para casos comunes.

`@EnableAutoConfiguration` es la anotación que activa la auto-configuración en una aplicación Spring Boot. Esta anotación se incluye típicamente a través de `@SpringBootApplication`, que combina `@Configuration`, `@ComponentScan` y `@EnableAutoConfiguration`. Cuando se usa `@SpringBootApplication`, la auto-configuración se habilita automáticamente sin necesidad de configuración adicional.

En Spring Boot 2.x, las auto-configuraciones se registraban en el archivo `META-INF/spring.factories`. Este archivo contenía una lista de clases de configuración que se cargaban condicionalmente durante el inicio de la aplicación. El archivo `spring.factories` utilizaba la propiedad `org.springframework.boot.autoconfigure.EnableAutoConfiguration` para listar las auto-configuraciones disponibles.

Spring Boot 3.x introdujo un mecanismo mejorado usando el archivo `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Este nuevo formato reemplaza a `spring.factories` para auto-configuraciones, proporcionando mejor rendimiento y claridad. El archivo contiene una lista simple de nombres de clases de configuración, una por línea, sin necesidad de propiedades adicionales.

Las anotaciones condicionales son fundamentales para el funcionamiento de la auto-configuración. Spring Boot utiliza anotaciones como `@ConditionalOnClass`, `@ConditionalOnBean`, `@ConditionalOnMissingBean` y `@ConditionalOnProperty` para determinar si una auto-configuración debe aplicarse. Por ejemplo, `@ConditionalOnClass(DataSource.class)` indica que la configuración solo se aplicará si la clase `DataSource` está presente en el classpath.

`@ConditionalOnMissingBean` es particularmente importante porque permite a los usuarios overridear la configuración por defecto. Si un usuario define su propia bean de un tipo específico, Spring Boot no configurará la bean por defecto. Esto proporciona flexibilidad mientras mantiene la configuración convencional para casos comunes.

```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

Para debugear la auto-configuración, Spring Boot proporciona varias opciones. El flag `--debug` en la línea de comandos activa el logging de auto-configuración, mostrando qué configuraciones se aplicaron y por qué. Alternativamente, se puede habilitar el logging a nivel DEBUG para el paquete `org.springframework.boot.autoconfigure`.

```bash
java -jar mi-aplicacion.jar --debug
```

```properties
# En application.properties
logging.level.org.springframework.boot.autoconfigure=DEBUG
```

El endpoint `/actuator/conditions` del Actuator proporciona información detallada sobre todas las auto-configuraciones que se aplicaron o no. Este endpoint muestra las condiciones que se cumplieron o fallaron para cada auto-configuración, lo que es extremadamente útil para diagnosticar problemas de configuración.

```yaml
# Ejemplo de salida del endpoint /actuator/conditions
conditions:
  matched:
    - name: DataSourceAutoConfiguration
      conditions:
        - name: DataSourceAutoConfiguration$DataSourceInitializerPostProcessor
          matched:
            - type: OnClassCondition
              message: required @ConditionalOnClass javax.sql.DataSource was present
  unmatched:
    - name: SecurityAutoConfiguration
      conditions:
        - name: SecurityAutoConfiguration
          missed:
            - type: OnClassCondition
              message: required @ConditionalOnClass org.springframework.security.config.annotation.web.configuration.EnableWebSecurity was not present
```

La auto-configuración se puede deshabilitar completamente mediante la propiedad `spring.autoconfigure.exclude` o usando la anotación `@EnableAutoConfiguration(exclude = {...})`. Esto es útil cuando se quiere control total sobre la configuración o cuando una auto-configuración entra en conflicto con configuraciones personalizadas.

> La auto-configuración no es magia: es código Java que utiliza anotaciones condicionales para decidir si configurar o no ciertas beans. Entender este mecanismo permite diagnosticar problemas y personalizar el comportamiento de Spring Boot de manera efectiva.

