# Spring Boot vs Spring Framework

La distincion entre Spring Framework y Spring Boot es una de las fuentes de confusion mas frecuentes entre desarrolladores que comienzan en el ecosistema Spring. Aunque ambos comparten el nombre y el nucleo tecnologico, resuelven problemas fundamentalmente diferentes. Spring Framework es el contenedor IoC y el conjunto de modulos que proporcionan la infraestructura basica para construir aplicaciones Java empresariales. Spring Boot es una capa de construccion sobre Spring Framework que elimina la configuracion manual y adopta un enfoque de "convention over configuration" (convencion sobre configuracion).

Sin Spring Boot, configurar una aplicacion Spring requiere multiples pasos manuales: definir un archivo `web.xml` o una clase `WebApplicationInitializer`, configurar el `DispatcherServlet`, establecer un `ViewResolver`, configurar un `EntityManagerFactory` si se usa JPA, definir un `DataSource` con los parametros de conexion, habilitar la deteccion de transacciones, y un largo etcetera. Cada uno de estos pasos implica clases de configuracion, archivos XML o anotaciones especificas que el desarrollador debe conocer y mantener.

Spring Boot aborda este problema mediante tres mecanismos fundamentales: auto-configuracion, starter dependencies y embedded servers.

## Auto-configuracion

La auto-configuracion de Spring Boot analiza el classpath de la aplicacion y configura automaticamente los beans que detecta como necesarios. Cuando Spring Boot encuentra la libreria Hibernate en el classpath, configura un `EntityManagerFactory` y un `DataSource` con valores por defecto razonables. Si encuentra Spring Security, configura un filtro de seguridad basico con autenticacion form-login. Si encuentra Thymeleaf, configura un `TemplateResolver` y un `ViewResolver` apuntando a `src/main/resources/templates`.

Detras de este mecanismo se encuentra la anotacion `@EnableAutoConfiguration`, que importa las definiciones de configuracion desde los archivos `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3.x) o `META-INF/spring.factories` (Spring Boot 2.x). Cada auto-configuracion es una clase Java anotada con `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty` y otras anotaciones condicionales que determinan si la configuracion debe aplicarse o no.

```java
// Ejemplo de auto-configuracion condicional
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnProperty(name = "spring.datasource.url")
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .username(properties.getUsername())
            .password(properties.getPassword())
            .build();
    }
}
```

La clave del exito de la auto-configuracion es que siempre es overridable. Si defines tu propio bean de `DataSource` en la configuracion de la aplicacion, Spring Boot no creara el suyo automatico. Esto permite un control total cuando es necesario, sin sacrificar la simplicidad en los casos comunes.

## Starter dependencies

Las starter dependencies de Spring Boot son agrupaciones de dependencias Maven o Gradle que encapsulan todos los modulos y librerias necesarios para un proposito especifico. En lugar de declarar dependencias individuales de `spring-core`, `spring-beans`, `spring-context`, `spring-aop`, `jackson-databind`, `tomcat-embed-core` y otros, se declara una unica dependencia:

```xml
<!-- En lugar de declarar multiples dependencias manualmente -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Acceso a datos con JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- PostgreSQL driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

Cada starter incluye las dependencias transitivas necesarias y.versiones compatibles entre si. Spring Boot gestiona las versiones de todas las librerias a traves de la "Bill of Materials" (BOM), eliminando conflictos de versiones entre dependencias.

## Embedded servers

Una de las diferencias mas significativas entre una aplicacion Spring tradicional y una aplicacion Spring Boot es la eliminacion de la necesidad de un servidor de aplicaciones externo. Las aplicaciones Spring tradicionales se empaquetan como archivos WAR que deben desplegarse en un contenedor servlet (Tomcat, Jetty, WildFly). Spring Boot, en cambio, empaqueta la aplicacion como un JAR ejecutable que incluye un servidor web embebido.

```bash
# Compilar y ejecutar una aplicacion Spring Boot
./mvnw clean package
java -jar target/mi-aplicacion.jar

# Ejecutar directamente durante el desarrollo
./mvnw spring-boot:run
```

Spring Boot soporta Tomcat (por defecto), Jetty y Undertow como servidores embebidos. La eleccion se realiza mediante la exclusion del starter por defecto y la inclusion del starter alternativo:

```xml
<!-- Usar Jetty en lugar de Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

> La decision de empaquetar el servidor dentro de la aplicacion fue controversia cuando Spring Boot se lanzo. La objecion principal era que violaba el modelo de despliegue tradicional donde el operador controlaba el servidor. Sin embargo, la practica demostro que las ventajas de simplicidad, portabilidad y consistencia de entornos superaron ampliamente las desventajas.

## Spring Boot Actuator

Spring Boot Actuator anade capacidades de monitoreo y gestion a la aplicacion sin configuracion adicional. Habilita endpoints HTTP que exponen informacion sobre el estado de la aplicacion, metricas de negocio y technicas, y controles de salud.

```yaml
# application.yml con Actuator configurado
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when-authorized
```

```bash
# Verificar el estado de la aplicacion
curl http://localhost:8080/actuator/health

# Obtener metricas de JVM
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Explosar el contexto para diagnostico
curl http://localhost:8080/actuator/beans
```

Los endpoints mas comunes incluyen `/health` (estado de la aplicacion y sus dependencias), `/info` (informacion del proyecto), `/metrics` (metricas de rendimiento), `/env` (propiedades de entorno), `/beans` (lista de beans cargados) y `/loggers` (configuracion de logging en tiempo real).

## Spring Initializr

Spring Initializr es una herramienta web (https://start.spring.io) y CLI que genera la estructura base de un proyecto Spring Boot con las dependencias seleccionadas. Genera un proyecto Maven o Gradle con la estructura de directorios estandar, la clase principal de la aplicacion y un archivo de configuracion basico.

```bash
# Crear un proyecto con Spring CLI
spring init --dependencies=web,data-jpa,postgresql mi-app

# Crear un proyecto con curl
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql \
  -d name=mi-app \
  -o mi-app.zip
```

## Configuracion manual vs Spring Boot: comparacion directa

Para entender la magnitud de la simplificacion que Spring Boot aporta, consideremos la configuracion de una aplicacion web con JPA y PostgreSQL.

**Sin Spring Boot (configuracion manual):**

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) {
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);
        context.setServletContext(servletContext);

        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}

@Configuration
@EnableWebMvc
@EnableTransactionManagement
@ComponentScan(basePackages = "com.example")
@PropertySource("classpath:application.properties")
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName(env.getProperty("db.driver"));
        ds.setUrl(env.getProperty("db.url"));
        ds.setUsername(env.getProperty("db.username"));
        ds.setPassword(env.getProperty("db.password"));
        return ds;
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource());
        em.setPackagesToScan("com.example.model");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        em.setJpaProperties(hibernateProperties());
        return em;
    }

    @Bean
    public PlatformTransactionManager transactionManager() {
        JpaTransactionManager txManager = new JpaTransactionManager();
        txManager.setEntityManagerFactory(entityManagerFactory().getObject());
        return txManager;
    }

    private Properties hibernateProperties() {
        Properties props = new Properties();
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.PostgreSQLDialect");
        props.setProperty("hibernate.hbm2ddl.auto", "update");
        props.setProperty("hibernate.show_sql", "true");
        return props;
    }
}
```

**Con Spring Boot:**

```java
@SpringBootApplication
public class MiAplicacion {
    public static void main(String[] args) {
        SpringApplication.run(MiAplicacion.class, args);
    }
}
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mi_db
    username: usuario
    password: contrasena
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

La diferencia es abismal. Spring Boot elimina la necesidad de configurar manualmente el `DispatcherServlet`, el `DataSource`, el `EntityManagerFactory`, el `TransactionManager` y multiples propiedades de Hibernate. Todo se configura automaticamente a partir de las propiedades del archivo `application.yml`.

> Spring Boot no es un reemplazo de Spring Framework, sino una extension. Toda la funcionalidad de Spring Framework sigue disponible y accesible. Spring Boot simplemente reduce la friccion para los casos de uso mas comunes, manteniendo la capacidad de personalizacion completa cuando es necesaria.

