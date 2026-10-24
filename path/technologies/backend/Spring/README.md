# Spring

## Índice

- [Introducción a Spring](#introducción-a-spring)
- [Spring Core: IoC y DI](#spring-core-ioc-y-di)
- [Spring Boot](#spring-boot)
- [Spring MVC](#spring-mvc)
- [Spring AOP](#spring-aop)
- [Gestión de transacciones](#gestión-de-transacciones)
- [Spring Data](#spring-data)
- [Spring Security](#spring-security)
- [Spring Batch](#spring-batch)
- [Spring Integration](#spring-integration)
- [Spring Cloud](#spring-cloud)
- [Spring WebFlux](#spring-webflux)
- [Spring Native y GraalVM](#spring-native-y-graalvm)
- [Spring Boot Actuator](#spring-boot-actuator)
- [Spring Boot y DevOps](#spring-boot-y-devops)
- [Buenas prácticas](#buenas-prácticas)

> [!WARNING]
> Los apuntes no tienen acentos ni ñ porque, al probar una extensión nueva con IA, los ha borrado. Cuando termine de subir todo, lo solucionaré.

## Introducción a Spring

- [Qué es Spring](01-introduccion_a_spring/01-que_es_spring.md)
- [Módulos de Spring](01-introduccion_a_spring/02-modulos_de_spring.md)
- [Spring Boot vs Spring Framework](01-introduccion_a_spring/03-spring_boot_vs_spring_framework.md)

## Spring Core: IoC y DI

- [Inversión de control](02-spring_core_ioc_y_di/01-inversion_de_control.md)
- [Inyección de dependencias](02-spring_core_ioc_y_di/02-dependency_injection.md)
- [Ciclo de vida del bean](02-spring_core_ioc_y_di/03-ciclo_de_vida_del_bean.md)
- [Scopes de beans](02-spring_core_ioc_y_di/04-scopes_de_beans.md)
- [Configuración](02-spring_core_ioc_y_di/05-configuration.md)
- [Profiles](02-spring_core_ioc_y_di/06-profiles.md)

## Spring Boot

- [Qué es Spring Boot](03-spring_boot/01-que_es_spring_boot.md)
- [Spring Initializr](03-spring_boot/02-spring_initializr.md)
- [Starter dependencies](03-spring_boot/03-starter_dependencies.md)
- [Autoconfiguración](03-spring_boot/04-auto_configuration.md)
- [Application properties](03-spring_boot/05-application_properties.md)
- [Logging](03-spring_boot/06-logging.md)
- [DevTools](03-spring_boot/07-devtools.md)
- [Actuator](03-spring_boot/08-actuator.md)
- [Packaging](03-spring_boot/09-packaging.md)
- [Spring Boot CLI](03-spring_boot/10-spring_boot_cli.md)

## Spring MVC

- [Qué es Spring MVC](04-spring_mvc/01-que_es_spring_mvc.md)
- [Controllers](04-spring_mvc/02-controllers.md)
- [Request y response](04-spring_mvc/03-request_response.md)
- [Validación](04-spring_mvc/04-validation.md)
- [Manejo de excepciones](04-spring_mvc/05-exception_handling.md)
- [Configuración de CORS](04-spring_mvc/06-cors_configuration.md)
- [Jackson y JSON](04-spring_mvc/07-jackson_json.md)
- [Clientes HTTP](04-spring_mvc/08-http_clients.md)
- [Soporte asíncrono](04-spring_mvc/09-async_support.md)

## Spring AOP

- [Qué es AOP](05-spring_aop/01-que_es_aop.md)
- [Fundamentos de Spring AOP](05-spring_aop/02-spring_aop_basics.md)
- [Expresiones pointcut](05-spring_aop/03-pointcut_expressions.md)
- [Proxies de AOP](05-spring_aop/04-aop_proxies.md)

## Gestión de transacciones

- [Qué es una transacción](06-spring_transaction_management/01-que_es_transaccion.md)
- [Anotación @Transactional](06-spring_transaction_management/02-anotacion_transactional.md)
- [Transacciones programáticas](06-spring_transaction_management/03-transacciones_programaticas.md)
- [Buenas prácticas de transacciones](06-spring_transaction_management/04-mejores_practicas.md)

## Spring Data

- [Qué es Spring Data](07-spring_data/01-que_es_spring_data.md)
- [Query methods](07-spring_data/02-query_methods.md)
- [Paginación y ordenación](07-spring_data/03-pagination_and_sorting.md)
- [Projections](07-spring_data/04-projections.md)
- [Auditing](07-spring_data/05-auditing.md)
- [Spring Data JPA](07-spring_data/06-spring_data_jpa.md)
- [Spring Data MongoDB](07-spring_data/07-spring_data_mongodb.md)
- [Spring Data Redis](07-spring_data/08-spring_data_redis.md)
- [Spring Data REST](07-spring_data/09-spring_data_rest.md)

## Spring Security

- [Qué es Spring Security](08-spring_security/01-que_es_spring_security.md)
- [Configuración básica](08-spring_security/02-basic_configuration.md)
- [Autenticación](08-spring_security/03-authentication.md)
- [Autorización](08-spring_security/04-authorization.md)
- [Form login](08-spring_security/05-form_login.md)
- [HTTP Basic](08-spring_security/06-http_basic.md)
- [OAuth 2 y OIDC](08-spring_security/07-oauth2_oidc.md)
- [JWT](08-spring_security/08-jwt.md)
- [Seguridad de métodos](08-spring_security/09-method_security.md)
- [Protección CSRF](08-spring_security/10-csrf_protection.md)
- [Gestión de sesiones](08-spring_security/11-session_management.md)
- [Buenas prácticas de seguridad](08-spring_security/12-security_best_practices.md)

## Spring Batch

- [Qué es Spring Batch](09-spring_batch/01-que_es_spring_batch.md)
- [Arquitectura](09-spring_batch/02-architecture.md)
- [Componentes](09-spring_batch/03-components.md)
- [Configuración](09-spring_batch/04-configuration.md)
- [Funciones avanzadas](09-spring_batch/05-advanced_features.md)

## Spring Integration

- [Qué es Spring Integration](10-spring_integration/01-que_es_spring_integration.md)
- [Componentes](10-spring_integration/02-components.md)
- [Adapters](10-spring_integration/03-adapters.md)
- [Configuración](10-spring_integration/04-configuration.md)

## Spring Cloud

- [Qué es Spring Cloud](11-spring_cloud/01-que_es_spring_cloud.md)
- [Service discovery](11-spring_cloud/02-service_discovery.md)
- [Config Server](11-spring_cloud/03-config_server.md)
- [API Gateway](11-spring_cloud/04-api_gateway.md)
- [Balanceo de carga](11-spring_cloud/05-load_balancing.md)
- [Circuit breaker](11-spring_cloud/06-circuit_breaker.md)
- [Trazabilidad distribuida](11-spring_cloud/07-distributed_tracing.md)
- [Mensajería](11-spring_cloud/08-messaging.md)
- [Pruebas de contratos](11-spring_cloud/09-contract_testing.md)

## Spring WebFlux

- [Qué es WebFlux](12-spring_webflux/01-que_es_webflux.md)
- [Modelo de programación](12-spring_webflux/02-programming_model.md)
- [WebClient](12-spring_webflux/03-webclient.md)
- [Backpressure](12-spring_webflux/04-backpressure.md)
- [Spring MVC vs WebFlux](12-spring_webflux/05-spring_mvc_vs_webflux.md)
- [Acceso reactivo a datos](12-spring_webflux/06-reactive_data_access.md)

## Spring Native y GraalVM

- [Qué es Spring Native](13-spring_native_graalvm/01-que_es_spring_native.md)
- [Configuración](13-spring_native_graalvm/02-configuration.md)
- [Limitaciones](13-spring_native_graalvm/03-limitaciones.md)

## Spring Boot Actuator

- [Health indicators](14-spring_boot_actuator/01-health_indicators.md)
- [Métricas](14-spring_boot_actuator/02-metrics.md)
- [Info](14-spring_boot_actuator/03-info.md)
- [Environment](14-spring_boot_actuator/04-environment.md)
- [Audit](14-spring_boot_actuator/05-audit.md)
- [Endpoints personalizados](14-spring_boot_actuator/06-custom_endpoints.md)

## Spring Boot y DevOps

- [Configuración externalizada](15-spring_boot_devops/01-externalized_configuration.md)
- [Configuración de producción](15-spring_boot_devops/02-production_configuration.md)
- [Monitorización](15-spring_boot_devops/03-monitoring.md)
- [Soporte para contenedores](15-spring_boot_devops/04-container_support.md)
- [Soporte para Kubernetes](15-spring_boot_devops/05-kubernetes_support.md)
- [Spring Boot Admin](15-spring_boot_devops/06-spring_boot_admin.md)

## Buenas prácticas

- [Estructura de paquetes](16-buenas_practicas/01-package_structure.md)
- [Gestión de configuración](16-buenas_practicas/02-configuration_management.md)
- [Estrategia de manejo de excepciones](16-buenas_practicas/03-exception_handling_strategy.md)
- [Estrategia de logging](16-buenas_practicas/04-logging_strategy.md)
- [Optimización del rendimiento](16-buenas_practicas/05-performance_optimization.md)
- [Estrategia de testing](16-buenas_practicas/06-testing_strategy.md)
- [Buenas prácticas de seguridad](16-buenas_practicas/07-security_best_practices.md)
- [Preparación para producción](16-buenas_practicas/08-production_readiness.md)