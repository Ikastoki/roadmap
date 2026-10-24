# DevTools

Spring Boot DevTools es un módulo que proporciona herramientas para mejorar la experiencia de desarrollo, particularmente durante el ciclo de codificación-compilación-prueba. Sus características principales incluyen restart automático de la aplicación cuando se detectan cambios en el classpath y soporte para live reload en el navegador.

Para habilitar DevTools, simplemente se agrega la dependencia al proyecto. Spring Boot detecta automáticamente la presencia de DevTools y ajusta su comportamiento en consecuencia. Durante el desarrollo, DevTools se carga automáticamente cuando la aplicación se ejecuta desde el IDE o con `mvn spring-boot:run`.

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

```groovy
// Gradle
developmentOnly 'org.springframework.boot:spring-boot-devtools'
```

El restart automático es la característica más conocida de DevTools. Cuando se detectan cambios en los archivos del classpath, la aplicación se reinicia automáticamente. DevTools implementa un classloader de dos niveles: uno para las bibliotecas base que no cambian frecuentemente y otro para el código de la aplicación que cambia constantemente. Solo el segundo classloader se reinicia, lo que reduce significativamente el tiempo de restart.

Para mejorar aún más el tiempo de restart, se puede habilitar el auto-save automático. Esto permite que los cambios se apliquen inmediatamente al guardar los archivos en el IDE, sin necesidad de ejecutar un rebuild manual. La configuración varía según el IDE: en IntelliJ IDEA se habilita en Settings > Build, Execution, Deployment > Compiler, mientras que en Eclipse se activa automáticamente.

Live Reload es otra característica importante que actualiza automáticamente el navegador cuando se detectan cambios en los recursos estáticos (HTML, CSS, JavaScript). DevTools incluye un plugin de navegador que se comunica con el servidor de desarrollo para recibir notificaciones de cambios. Este plugin está disponible para Chrome, Firefox y Safari.

```properties
# Configuración de DevTools en application.properties
spring.devtools.restart.enabled=true
spring.devtools.restart.exclude=static/**,public/**
spring.devtools.restart.additional-paths=src/main/resources
spring.devtools.livereload.enabled=true
```

Las configuraciones globales de DevTools se pueden establecer en `~/.spring-boot-devtools.properties`. Estas configuraciones se aplican a todas las aplicaciones Spring Boot en la máquina, lo que es útil para establecer preferencias de desarrollo consistentes. Las propiedades globales se prefixean con `spring.devtools.`.

```properties
# ~/.spring-boot-devtools.properties
spring.devtools.restart.trigger-file=.reloadtrigger
spring.devtools.restart.additional-exclude=test/**,build/**
```

El restart se puede deshabilitar completamente mediante la propiedad `spring.devtools.restart.enabled=false`. Esto es necesario en producción, donde DevTools no debe estar presente. De hecho, DevTools automáticamente se desactiva cuando la aplicación se ejecuta como JAR empaquetado, ya que detecta que no está en modo desarrollo.

Hay varias limitaciones conocidas que se deben tener en cuenta al usar DevTools. Los cambios en beans con `@ConfigurationProperties` pueden no reiniciar la aplicación inmediatamente. Los cambios en recursos estáticos no triggerean un restart pero sí actualizan el live reload. Los cambios en JPA entities y relaciones de persistencia pueden requerir un restart manual debido a la complejidad de las configuraciones de persistencia.

El trigger file es una característica que permite personalizar cuándo se activa el restart. En lugar de reiniciar ante cada cambio en el classpath, se puede configurar DevTools para que solo reinicie cuando cambie un archivo específico. Esto es útil en entornos donde se realizan muchos cambios rápidos y se quiere controlar cuándo se aplican.

```properties
# Configurar trigger file
spring.devtools.restart.trigger-file=.reloadtrigger
spring.devtools.restart.additional-trigger-files=src/main/resources/.reloadtrigger
```

> DevTools no debe usarse nunca en producción. La dependencia se debe marcar como `runtime` o `optional` para asegurar que no se incluya en el JAR de producción. Spring Boot automáticamente deshabilita DevTools cuando detecta que la aplicación se ejecuta desde un JAR empaquetado.

