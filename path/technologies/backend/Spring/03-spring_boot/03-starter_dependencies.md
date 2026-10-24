# Starter Dependencies

Spring Boot Starter Dependencies son conjuntos de dependencias que incluyen todas las bibliotecas necesarias para funcionalidades específicas del framework. Estos starters eliminan la necesidad de configurar múltiples dependencias individuales y gestionar versiones compatibles, proporcionando una forma simplificada de agregar capacidades a la aplicación.

El starter base `spring-boot-starter` es la dependencia fundamental que incluye el core de Spring Boot, logging y auto-configuración básica. Toda aplicación Spring Boot debe incluir este starter, aunque generalmente se incluye automáticamente a través de otros starters más específicos.

`spring-boot-starter-web` es uno de los starters más utilizados, proporcionando todo lo necesario para desarrollo web con Spring MVC. Incluye Spring MVC, Tomcat como servidor embebido, Jackson para serialización JSON y validación de beans. Este starter configura automáticamente un servidor web completo con todas las dependencias necesarias para aplicaciones REST y MVC.

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```groovy
// Gradle
implementation 'org.springframework.boot:spring-boot-starter-web'
```

`spring-boot-starter-data-jpa` agrega soporte para persistencia de datos usando JPA (Java Persistence API) con Hibernate como implementación por defecto. Incluye Spring Data JPA, Hibernate, DataSource y transacciones管理. Este starter configura automáticamente un EntityManagerFactory y un TransactionManager, eliminando la necesidad de configuración manual de JPA.

`spring-boot-starter-security` proporciona soporte completo de seguridad para aplicaciones Spring. Incluye Spring Security con configuración por defecto para autenticación básica, prevención de ataques CSRF, protección contra XSS y configuración de headers de seguridad. La auto-configuración crea un usuario por defecto con credenciales aleatorias si no se proporciona ninguna configuración adicional.

`spring-boot-starter-test` es el starter de testing que incluye JUnit 5, Mockito, Spring Test y AssertJ para pruebas unitarias e de integración. Proporciona utilidades como `@SpringBootTest` que carga el contexto completo de la aplicación para pruebas de integración, y `@WebMvcTest` para pruebas de capa web aisladas.

`spring-boot-starter-actuator` agrega capacidades de monitoreo y métricas a la aplicación. Incluye endpoints como `/health`, `/info`, `/metrics` y `/beans` que permiten inspeccionar el estado de la aplicación. También integra con Micrometer para métricas y observabilidad, soportando múltiples sistemas de monitoreo como Prometheus, Datadog y New Relic.

`spring-boot-starter-validation` proporciona soporte para Bean Validation (JSR 380) usando Hibernate Validator. Incluye anotaciones como `@NotNull`, `@Size`, `@Email` y permite crear validadores personalizados. Este starter es esencial para validar datos de entrada en controladores REST y servicios.

`spring-boot-starter-cache` agrega soporte para caching en la aplicación. Incluye abstracciones de Spring Cache con implementaciones por defecto como ConcurrentHashMap y soporte para sistemas de caching avanzados como EhCache, Caffeine y Redis. La configuración se realiza mediante anotaciones como `@Cacheable` y `@CacheEvict`.

`spring-boot-starter-amqp` proporciona soporte para mensajería con RabbitMQ. Incluye Spring AMQP para enviar y recibir mensajes, configuración automática de conexiones y colas, y soporte para patrones como publish-subscribe y punto a punto. Este starter es fundamental para arquitecturas orientadas a eventos y microservicios.

`spring-boot-starter-kafka` agrega soporte para Apache Kafka, el sistema de mensajería distribuida. Incluye Spring Kafka para producción y consumo de mensajes, configuración de productores y consumidores, y soporte para patrones como event sourcing y CQRS. Ideal para aplicaciones de alto rendimiento que requieren procesamiento de streams.

`spring-boot-starter-data-redis` proporciona soporte para Redis como almacenamiento de datos. Incluye Spring Data Redis con soporte para operaciones de clave-valor, estructuras de datos como hashes y listas, y patrones como caching distribuido y sesiones. La auto-configuración establece conexiones a Redis y proporciona templates listos para usar.

Los custom starters permiten crear dependencias reutilizables que encapsulan configuraciones específicas de un proyecto o equipo. Un custom starter típicamente contiene dos módulos: un módulo de auto-configuración que define las beans y configuraciones, y un módulo de dependencias que agrega las bibliotecas necesarias. Esto facilita la compartición de configuraciones comunes entre múltiples proyectos.

La estructura de un custom starter sigue convenciones específicas. El módulo de auto-configuración debe seguir el paquete `org.springframework.boot.autoconfigure` y contener una clase anotada con `@AutoConfiguration`. Esta clase utiliza anotaciones condicionales para determinar cuándo aplicar la configuración. El módulo de dependencias simplemente agrega las bibliotecas necesarias y referencia al módulo de auto-configuración.

```xml
<!-- Estructura de un custom starter -->
<project>
    <artifactId>mi-company-spring-boot-starter</artifactId>
    
    <modules>
        <module>mi-company-spring-boot-autoconfigure</module>
        <module>mi-company-spring-boot-starter</module>
    </modules>
</project>

<!-- En el módulo de dependencias -->
<dependencies>
    <dependency>
        <groupId>com.mi.empresa</groupId>
        <artifactId>mi-company-spring-boot-autoconfigure</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

La gestión de versiones de los starters es crítica para evitar conflictos. Spring Boot administra las versiones de todas las dependencias mediante la BOM (Bill of Materials) `spring-boot-dependencies`. Esta BOM define versiones compatibles para cientos de bibliotecas, eliminando la necesidad de especificar versiones individuales en los proyectos. Al usar starters, las versiones se heredan automáticamente de la BOM.

Es importante comprender que los starters no son la única forma de agregar dependencias a un proyecto Spring Boot. Se pueden agregar dependencias individuales de Maven o Gradle directamente. Sin embargo, los starters ofrecen ventajas significativas: agrupan dependencias relacionadas, incluyen auto-configuración y son probados juntos para garantizar compatibilidad. Esta combinación reduce la posibilidad de conflictos y simplifica la gestión de dependencias.

Cuando se necesita excluir una dependencia específica de un starter, se puede hacer mediante las exclusiones de Maven o Gradle. Esto es útil cuando el starter incluye una biblioteca que no se necesita o que entra en conflicto con otra versión ya presente en el proyecto. La exclusión se realiza a nivel de dependencia, no a nivel de starter.

> Los starter dependencies no solo agregan bibliotecas, sino que también incluyen auto-configuraciones que configuran automáticamente las beans necesarias. Esto significa que al agregar un starter, la funcionalidad correspondiente está lista para usar sin configuración adicional.

