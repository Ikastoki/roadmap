# Spring Boot Admin

Spring Boot Admin es una aplicacion web que proporciona una interfaz grafica para monitorizar y gestionar una o varias aplicaciones Spring Boot. Se comunica con los endpoints de Actuator de cada aplicacion registrada y presenta la informacion de salud, metricas, logs, env y beans de forma visual e interactiva. Es una herramienta especialmente util en entornos con multiples microservicios donde se necesita una vista centralizada del estado del sistema.

## Arquitectura

Spring Boot Admin funciona como un cliente-servidor. El servidor (`spring-boot-admin-server`) expone un dashboard web que recopila informacion de multiples aplicaciones clientes (`spring-boot-admin-client`). La comunicacion se realiza a traves de los endpoints de Actuator, y las aplicaciones clientes se registran automaticamente con el servidor.

```xml
<!-- Servidor Spring Boot Admin -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>3.1.0</version>
</dependency>
```

```java
@SpringBootApplication
@EnableAdminServer
public class AdminServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }
}
```

```yaml
# Configuracion del servidor
server:
  port: 8400

spring:
  application:
    name: spring-boot-admin
```

## Registro de aplicaciones

Las aplicaciones clientes se registran con el servidor de administracion de forma automatica usando el starter de cliente. El cliente periodicamente reporta su estado y metricas al servidor.

```xml
<!-- En la aplicacion cliente -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>3.1.0</version>
</dependency>
```

```yaml
spring:
  boot:
    admin:
      client:
        url: http://localhost:8400
        instance:
          prefer-ip: true
          metadata:
            user: admin
            password: ${management.endpoints.web.base-path:}/actuator
        auto-registration: true
        period: 10s

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,loggers,heapdump,threaddump
  endpoint:
    health:
      show-details: always
```

Para el registro manual o con discovery, se puede utilizar Kubernetes, Eureka o una lista estatica de URLs:

```yaml
spring:
  boot:
    admin:
      client:
        url:
          - http://servicio-1:8080
          - http://servicio-2:8080
          - http://servicio-3:8080
```

## Funcionalidades del dashboard

Spring Boot Admin proporciona las siguientes secciones principales en su interfaz web:

- **Health**: estado de cada aplicacion con detalles de los health indicators configurados
- **Metrics**: graficas de metricas Micrometer (HTTP, JVM, system) con filtros por tiempo
- **Environment**: explorador de propiedades del entorno con busqueda y filtrado
- **Loggers**: visualizacion y ajuste dinamico de niveles de log sin reiniciar la aplicacion
- **Threads**: dump de hilos activos con analisis de estado
- **Heap**: descarga y analisis de heap dumps para diagnostico de memoria
- **Mappings**: listado de endpoints HTTP registrados con sus controllers y handlers
- **Beans**: arbol completo de beans de Spring con detalles de inyeccion y dependencias

```yaml
# Habilitar endpoints especificos para Spring Boot Admin
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,loggers,heapdump,threaddump,mappings,beans
  endpoint:
    loggers:
      enabled: true
```

## Notificaciones

Spring Boot Admin soporta notificaciones automaticas cuando el estado de una aplicacion cambia. Las notificaciones se pueden configurar para enviar emails, Slack, Telegram o webhooks personalizados cuando una aplicacion pasa de UP a DOWN o viceversa.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

```yaml
spring:
  boot:
    admin:
      notify:
        mail:
          enabled: true
          to: admin@ejemplo.com
          from: spring-boot-admin@ejemplo.com
          subject: "[SBA] Cambio de estado: {{application.name}}"
          template: classpath:/mail/status-change.html

spring:
  mail:
    host: smtp.ejemplo.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
```

> Las notificaciones de Spring Boot Admin son asincronas y no bloquean el ciclo de vida de la aplicacion. En entornos de alta disponibilidad, se recomienda combinarlas con alertas de Prometheus/Grafana para una cobertura completa.

## Seguridad

El dashboard de Spring Boot Admin debe protegerse con autenticacion y autorizacion en entornos productivos. Spring Security puede integrarse facilmente para restringir el acceso.

```java
@Configuration
public class AdminSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/assets/**").permitAll()
                .requestMatchers("/login").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form.loginPage("/login"))
            .logout(logout -> logout.logoutUrl("/logout"))
            .csrf(csrf -> csrf.disable());

        return http.build();
    }

    @Bean
    public InMemoryUserDetailsManager userDetailsService() {
        UserDetails user = User.withDefaultPasswordEncoder()
            .username("admin")
            .password("admin")
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(user);
    }
}
```

La combinacion de Spring Boot Admin con Actuator proporciona una plataforma completa de monitorizacion y gestion que permite a los equipos operativos identificar problemas, ajustar configuraciones y diagnosticar incidentes sin necesidad de acceder a cada instancia individualmente.

