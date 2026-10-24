<h1>Java</h1>

<h2>Índice</h2>

- [Java Core](#java-core)
  - [Conceptos](#conceptos)
  - [Principios Basicos](#principios-basicos)
  - [Entrada Y Salida](#entrada-y-salida)
  - [Control De Flujo](#control-de-flujo)
  - [Metodos](#metodos)
  - [Arrays](#arrays)
  - [Poo](#poo)
  - [Clases Integradas](#clases-integradas)
  - [Colecciones](#colecciones)
  - [Ampliando Conceptos](#ampliando-conceptos)
  - [Programacion Funcional](#programacion-funcional)
  - [Hilos Y Concurrencia](#hilos-y-concurrencia)
  - [Patrones De Diseno](#patrones-de-diseno)
- [Java Capas](#java-capas)
  - [Fundamentos](#fundamentos)
  - [Capas Tipicas](#capas-tipicas)
  - [Convenciones Paquetes](#convenciones-paquetes)
  - [Capa Presentacion](#capa-presentacion)
  - [Capa Negocio](#capa-negocio)
  - [Capa Persistencia](#capa-persistencia)
  - [Patron Dto](#patron-dto)
  - [Variantes Arquitectura](#variantes-arquitectura)
  - [Patron Repository](#patron-repository)
  - [Patrones Complementarios](#patrones-complementarios)
  - [Testing](#testing)
  - [Implementacion Practica](#implementacion-practica)
  - [Buenas Practicas](#buenas-practicas)
- [Java Persistencia](#java-persistencia)
  - [Fundamentos De Persistencia](#fundamentos-de-persistencia)
  - [Jdbc](#jdbc)
  - [Jdbc Avanzado](#jdbc-avanzado)
  - [Orm](#orm)
  - [Jpa](#jpa)
  - [Hibernate](#hibernate)
  - [H2 Database](#h2-database)
  - [Connection Pooling](#connection-pooling)
  - [Transacciones](#transacciones)
  - [Spring Data Jpa](#spring-data-jpa)
  - [Testing De Persistencia](#testing-de-persistencia)
  - [Rendimiento De Persistencia](#rendimiento-de-persistencia)
  - [Manejo De Errores](#manejo-de-errores)
  - [Seguridad En Persistencia](#seguridad-en-persistencia)
  - [Buenas Practicas](#buenas-practicas-1)
- [Java Redes](#java-redes)
  - [Fundamentos De Redes](#fundamentos-de-redes)
  - [Sockets Tcp](#sockets-tcp)
  - [Sockets Udp](#sockets-udp)
  - [Io Clasica Vs Nio](#io-clasica-vs-nio)
  - [Nio 2](#nio-2)
  - [Http Con Java](#http-con-java)
  - [Httpclient Moderno](#httpclient-moderno)
  - [Urls Y Uris](#urls-y-uris)
  - [Sockets Nio](#sockets-nio)
  - [Programacion Asincrona](#programacion-asincrona)
  - [Threads Y Redes](#threads-y-redes)
  - [Apis Rest](#apis-rest)
  - [Seguridad En Redes](#seguridad-en-redes)
  - [Serializacion](#serializacion)
  - [Streaming](#streaming)
  - [Testing Y Herramientas](#testing-y-herramientas)
  - [Buenas Practicas](#buenas-practicas-2)
- [Java Testing](#java-testing)
  - [Fundamentos De Testing](#fundamentos-de-testing)
  - [Junit 5](#junit-5)
  - [Mockito](#mockito)
  - [Assertj](#assertj)
  - [Hamcrest](#hamcrest)
  - [Tdd](#tdd)
  - [Test Patterns](#test-patterns)
  - [Coverage De Codigo](#coverage-de-codigo)
  - [Property Testing](#property-testing)
  - [Performance Testing](#performance-testing)
  - [Configuracion Del Proyecto](#configuracion-del-proyecto)
  - [Manejo De Excepciones](#manejo-de-excepciones)
  - [Testing De Codigo Legado](#testing-de-codigo-legado)
  - [Testing En La Practica](#testing-en-la-practica)


> [!WARNING]
> Los apuntes no tienen acentos ni ñ porque, al probar una extensión nueva con IA, los ha borrado. Cuando termine de subir todo, lo solucionaré.

## Java Core

### Conceptos

- [Java](01-Java%20Core/01-conceptos/01-java.md)
- [Instalacion](01-Java%20Core/01-conceptos/02-instalacion.md)
- [Glosario](01-Java%20Core/01-conceptos/03-glosario.md)
- [Palabras reservadas](01-Java%20Core/01-conceptos/04-palabras_reservadas.md)
- [Compilacion y ejecucion](01-Java%20Core/01-conceptos/05-compilacion_y_ejecucion.md)
- [Bibliotecas principales](01-Java%20Core/01-conceptos/06-bibliotecas_principales.md)
- [Convencion de nombres](01-Java%20Core/01-conceptos/07-convencion_de_nombres.md)
- [Entrada principal](01-Java%20Core/01-conceptos/08-entrada_principal.md)

### Principios Basicos

- [Variables](01-Java%20Core/02-principios_basicos/01-variables.md)
- [Tipos de datos](01-Java%20Core/02-principios_basicos/02-tipos_de_datos.md)
- [Constantes](01-Java%20Core/02-principios_basicos/03-constantes.md)
- [Conversion de tipo](01-Java%20Core/02-principios_basicos/04-conversion_de_tipo.md)
- [Operadores](01-Java%20Core/02-principios_basicos/05-operadores.md)
- [Comentarios](01-Java%20Core/02-principios_basicos/06-comentarios.md)

### Entrada Y Salida

- [Imprimir datos](01-Java%20Core/03-entrada_y_salida/01-imprimir_datos.md)
- [Formatear salida](01-Java%20Core/03-entrada_y_salida/02-formatear_salida.md)
- [Scanner](01-Java%20Core/03-entrada_y_salida/03-scanner.md)
- [Java IO](01-Java%20Core/03-entrada_y_salida/04-java_io.md)

### Control De Flujo

- [Condicionales](01-Java%20Core/04-control_de_flujo/01-condicionales.md)
- [Bucles](01-Java%20Core/04-control_de_flujo/02-bucles.md)
- [Iteradores](01-Java%20Core/04-control_de_flujo/03-iteradores.md)
- [Continue, break, return](01-Java%20Core/04-control_de_flujo/04-continue_break_return.md)
- [Excepciones](01-Java%20Core/04-control_de_flujo/05-excepciones.md)

### Metodos

- [Metodos](01-Java%20Core/05-metodos/01-metodos.md)
- [Tipos de retorno](01-Java%20Core/05-metodos/02-tipos_de_retorno.md)
- [Parametros](01-Java%20Core/05-metodos/03-parametros.md)
- [Modificadores de acceso](01-Java%20Core/05-metodos/04-modificadores_de_acceso.md)
- [This](01-Java%20Core/05-metodos/05-this.md)
- [Sobrecarga](01-Java%20Core/05-metodos/06-sobrecarga.md)
- [Sobreescritura](01-Java%20Core/05-metodos/07-sobreescritura.md)
- [Pila de llamadas](01-Java%20Core/05-metodos/08-pila_de_llamadas.md)

### Arrays

- [Arrays](01-Java%20Core/06-arrays/01-arrays.md)
- [Arrays multidimensionales](01-Java%20Core/06-arrays/02-multidimensional.md)
- [Clase Array](01-Java%20Core/06-arrays/03-clase_array.md)
- [Arrays irregulares](01-Java%20Core/06-arrays/04-arrays_irregulares.md)
- [Final](01-Java%20Core/06-arrays/05-final.md)

### Poo

- [POO](01-Java%20Core/07-poo/01-poo.md)
- [Introduccion a clases y objetos](01-Java%20Core/07-poo/02-introduccion_a_clases.md)
- [Constructores](01-Java%20Core/07-poo/03-constructores.md)
- [Encapsulacion](01-Java%20Core/07-poo/04-encapsulacion.md)
- [Abstraccion](01-Java%20Core/07-poo/05-abstraccion.md)
- [Herencia](01-Java%20Core/07-poo/06-herencia.md)
- [Polimorfismo](01-Java%20Core/07-poo/07-polimorfismo.md)
- [Clase Object](01-Java%20Core/07-poo/08-clase_object.md)
- [Modificadores de acceso en POO](01-Java%20Core/07-poo/09-modificadores_de_acceso.md)
- [Paquetes](01-Java%20Core/07-poo/10-paquetes.md)
- [Static](01-Java%20Core/07-poo/11-static.md)

### Clases Integradas

- [String](01-Java%20Core/08-clases_integradas/01-string.md)
- [Math](01-Java%20Core/08-clases_integradas/02-math.md)
- [System](01-Java%20Core/08-clases_integradas/03-system.md)
- [Old Date (Calendar)](01-Java%20Core/08-clases_integradas/04-old_date_calendar.md)
- [Date (Time)](01-Java%20Core/08-clases_integradas/05-date_time.md)
- [File](01-Java%20Core/08-clases_integradas/06-file.md)

### Colecciones

- [Listas](01-Java%20Core/09-colecciones/01-listas.md)
- [Conjuntos](01-Java%20Core/09-colecciones/02-conjuntos.md)
- [Mapas](01-Java%20Core/09-colecciones/03-mapas.md)
- [Colas](01-Java%20Core/09-colecciones/04-colas.md)
- [Colecciones sincronizadas](01-Java%20Core/09-colecciones/05-sincronizadas.md)
- [Otras colecciones](01-Java%20Core/09-colecciones/06-otras_colecciones.md)

### Ampliando Conceptos

- [Enum](01-Java%20Core/10-ampliando_conceptos/01-enum.md)
- [Serializacion](01-Java%20Core/10-ampliando_conceptos/02-serializacion.md)
- [Genericos](01-Java%20Core/10-ampliando_conceptos/03-genericos.md)
- [Anotaciones](01-Java%20Core/10-ampliando_conceptos/04-anotaciones.md)
- [Record](01-Java%20Core/10-ampliando_conceptos/05-record.md)
- [Functional Interfaces](01-Java%20Core/10-ampliando_conceptos/06-functional_interfaces.md)
- [Reflection](01-Java%20Core/10-ampliando_conceptos/07-reflection.md)

### Programacion Funcional

- [Programacion funcional](01-Java%20Core/11-programacion_funcional/01-programacion_funcional.md)
- [Lambdas](01-Java%20Core/11-programacion_funcional/02-lambdas.md)
- [Streams](01-Java%20Core/11-programacion_funcional/03-streams.md)
- [Optional](01-Java%20Core/11-programacion_funcional/04-optional.md)

### Hilos Y Concurrencia

- [Concurrencia](01-Java%20Core/12-hilos_y_concurrencia/01-concurrencia.md)
- [Hilos](01-Java%20Core/12-hilos_y_concurrencia/02-hilos.md)

### Patrones De Diseno

- [Patrones de diseno](01-Java%20Core/13-patrones_de_diseno/01-patrones_de_diseno.md)
- [Singleton](01-Java%20Core/13-patrones_de_diseno/02-singleton.md)
- [Factory](01-Java%20Core/13-patrones_de_diseno/03-factory.md)
- [Observer](01-Java%20Core/13-patrones_de_diseno/04-observer.md)
- [Decorator](01-Java%20Core/13-patrones_de_diseno/05-decorator.md)
- [Composite](01-Java%20Core/13-patrones_de_diseno/06-composite.md)

## Java Capas

### Fundamentos

- [Que es la arquitectura de capas](02-Java%20Capas/01-fundamentos/01-que_es_arquitectura_capas.md)
- [Beneficios de la arquitectura de capas](02-Java%20Capas/01-fundamentos/02-beneficios.md)
- [Limitaciones de la arquitectura de capas](02-Java%20Capas/01-fundamentos/03-limitaciones.md)
- [Principios generales de la arquitectura de capas](02-Java%20Capas/01-fundamentos/04-principios_generales.md)

### Capas Tipicas

- [Capas de la arquitectura](02-Java%20Capas/02-capas_tipicas/01-capas_de_la_arquitectura.md)
- [Responsabilidades por capa](02-Java%20Capas/02-capas_tipicas/02-responsabilidades_por_capa.md)

### Convenciones Paquetes

- [Estructura de paquetes](02-Java%20Capas/03-convenciones_paquetes/01-estructura_paquetes.md)
- [Convenciones de nomenclatura](02-Java%20Capas/03-convenciones_paquetes/02-naming_conventions.md)

### Capa Presentacion

- [Controller](02-Java%20Capas/04-capa_presentacion/01-controller.md)
- [DTO (Data Transfer Object)](02-Java%20Capas/04-capa_presentacion/02-dto.md)
- [Validacion de entrada](02-Java%20Capas/04-capa_presentacion/03-validacion.md)
- [Formateo de respuestas](02-Java%20Capas/04-capa_presentacion/04-respuestas.md)

### Capa Negocio

- [Service](02-Java%20Capas/05-capa_negocio/01-service.md)
- [Domain Model](02-Java%20Capas/05-capa_negocio/02-domain_model.md)
- [Patron Repository (concepto)](02-Java%20Capas/05-capa_negocio/03-repository_concepto.md)
- [Excepciones de negocio](02-Java%20Capas/05-capa_negocio/04-excepciones_negocio.md)

### Capa Persistencia

- [Entity y JPA](02-Java%20Capas/06-capa_persistencia/01-entity_jpa.md)
- [Repository (implementacion)](02-Java%20Capas/06-capa_persistencia/02-repository_impl.md)
- [DAO vs Repository](02-Java%20Capas/06-capa_persistencia/03-dao_vs_repository.md)
- [Pool de conexiones](02-Java%20Capas/06-capa_persistencia/04-pool_conexiones.md)

### Patron Dto

- [Tipos de DTO](02-Java%20Capas/07-patron_dto/01-tipos_dto.md)
- [Mappers](02-Java%20Capas/07-patron_dto/02-mappers.md)
- [Buenas practicas con DTOs](02-Java%20Capas/07-patron_dto/03-best_practices.md)

### Variantes Arquitectura

- [Arquitectura Hexagonal (Puertos y Adaptadores)](02-Java%20Capas/08-variantes_arquitectura/01-hexagonal.md)
- [Arquitectura Onion (Cebolla)](02-Java%20Capas/08-variantes_arquitectura/02-onion.md)
- [Clean Architecture](02-Java%20Capas/08-variantes_arquitectura/03-clean_architecture.md)
- [Comparativa de variantes arquitectonicas](02-Java%20Capas/08-variantes_arquitectura/04-comparativa.md)

### Patron Repository

- [Repository generico](02-Java%20Capas/09-patron_repository/01-repository_generico.md)
- [Specification pattern](02-Java%20Capas/09-patron_repository/02-specification_pattern.md)
- [Unit of Work](02-Java%20Capas/09-patron_repository/03-unit_of_work.md)

### Patrones Complementarios

- [Service Layer](02-Java%20Capas/10-patrones_complementarios/01-service_layer.md)
- [Data Mapper vs Active Record](02-Java%20Capas/10-patrones_complementarios/02-data_mapper_vs_active_record.md)
- [Facade](02-Java%20Capas/10-patrones_complementarios/03-facade.md)
- [Proxy](02-Java%20Capas/10-patrones_complementarios/04-proxy.md)

### Testing

- [Testing por capa](02-Java%20Capas/11-testing/01-testing_por_capa.md)
- [Testing de repositorios](02-Java%20Capas/11-testing/02-testing_repositorios.md)
- [Testing de servicios](02-Java%20Capas/11-testing/03-testing_servicios.md)
- [Testing de controllers](02-Java%20Capas/11-testing/04-testing_controllers.md)

### Implementacion Practica

- [Estructura del proyecto](02-Java%20Capas/12-implementacion_practica/01-estructura_proyecto.md)
- [Ejemplo completo](02-Java%20Capas/12-implementacion_practica/02-ejemplo_completo.md)
- [Casos de uso](02-Java%20Capas/12-implementacion_practica/03-casos_de_uso.md)

### Buenas Practicas

- [Principios SOLID aplicados a las capas](02-Java%20Capas/13-buenas_practicas/01-solid.md)
- [Anti-patrones en la arquitectura de capas](02-Java%20Capas/13-buenas_practicas/02-anti-patrones.md)
- [Convenciones de codigo en proyectos por capas](02-Java%20Capas/13-buenas_practicas/03-convenciones_codigo.md)
- [Seguridad en la arquitectura por capas](02-Java%20Capas/13-buenas_practicas/04-seguridad.md)

## Java Persistencia

### Fundamentos De Persistencia

- [Que es persistencia](03-Java%20Persistencia/01-fundamentos_de_persistencia/01-que_es_persistencia.md)
- [Tecnologias de persistencia en Java](03-Java%20Persistencia/01-fundamentos_de_persistencia/02-tecnologias_de_persistencia.md)
- [Modelo relacional vs objetos](03-Java%20Persistencia/01-fundamentos_de_persistencia/03-modelo_relacional_vs_objetos.md)

### Jdbc

- [Conceptos basicos de JDBC](03-Java%20Persistencia/02-jdbc/01-conceptos_basicos_de_jdbc.md)
- [Conexion a la base de datos](03-Java%20Persistencia/02-jdbc/02-conexion_a_base_de_datos.md)
- [Statements](03-Java%20Persistencia/02-jdbc/03-statements.md)
- [ResultSet](03-Java%20Persistencia/02-jdbc/04-resultset.md)
- [CRUD con JDBC](03-Java%20Persistencia/02-jdbc/05-crud_con_jdbc.md)
- [Manejo de errores y recursos](03-Java%20Persistencia/02-jdbc/06-manejo_de_errores_y_recursos.md)

### Jdbc Avanzado

- [Transacciones con JDBC](03-Java%20Persistencia/03-jdbc_avanzado/01-transacciones_con_jdbc.md)
- [Batch Processing](03-Java%20Persistencia/03-jdbc_avanzado/02-batch_processing.md)
- [Pool de conexiones](03-Java%20Persistencia/03-jdbc_avanzado/03-pool_de_conexiones.md)
- [Metadata de la base de datos](03-Java%20Persistencia/03-jdbc_avanzado/04-metadata_de_base_de_datos.md)
- [RowSet](03-Java%20Persistencia/03-jdbc_avanzado/05-rowset.md)

### Orm

- [Conceptos de ORM](03-Java%20Persistencia/04-orm/01-conceptos_de_orm.md)
- [Ventajas y desventajas de ORM](03-Java%20Persistencia/04-orm/02-ventajas_y_desventajas.md)
- [Patrones de persistencia](03-Java%20Persistencia/04-orm/03-patrones_de_persistencia.md)

### Jpa

- [Que es JPA](03-Java%20Persistencia/05-jpa/01-que_es_jpa.md)
- [Configuracion de JPA](03-Java%20Persistencia/05-jpa/02-configuracion_de_jpa.md)
- [Anotaciones basicas](03-Java%20Persistencia/05-jpa/03-anotaciones_basicas.md)
- [Mapeo de tipos](03-Java%20Persistencia/05-jpa/04-mapeo_de_tipos.md)
- [Relaciones](03-Java%20Persistencia/05-jpa/05-relaciones.md)
- [Consultas con JPA](03-Java%20Persistencia/05-jpa/06-consultas_con_jpa.md)
- [Gestion de entidades](03-Java%20Persistencia/05-jpa/07-gestion_de_entidades.md)

### Hibernate

- [Que es Hibernate](03-Java%20Persistencia/06-hibernate/01-que_es_hibernate.md)
- [Configuracion de Hibernate](03-Java%20Persistencia/06-hibernate/02-configuracion_de_hibernate.md)
- [Anotaciones Hibernate](03-Java%20Persistencia/06-hibernate/03-anotaciones_hibernate.md)
- [HQL](03-Java%20Persistencia/06-hibernate/04-hql.md)
- [Criteria API de Hibernate](03-Java%20Persistencia/06-hibernate/05-criteria_api_hibernate.md)
- [Cache en Hibernate](03-Java%20Persistencia/06-hibernate/06-cache_en_hibernate.md)
- [Callbacks y Eventos](03-Java%20Persistencia/06-hibernate/07-callbacks_y_eventos.md)
- [Generadores de identidad](03-Java%20Persistencia/06-hibernate/08-generadores_de_identidad.md)
- [Bloqueo y concurrencia](03-Java%20Persistencia/06-hibernate/09-bloqueo_y_concurrencia.md)
- [Mapeo avanzado](03-Java%20Persistencia/06-hibernate/10-mapeo_avanzado.md)
- [SQL nativo en Hibernate](03-Java%20Persistencia/06-hibernate/11-sql_nativo_en_hibernate.md)
- [Hibernate en Spring Boot](03-Java%20Persistencia/06-hibernate/12-hibernate_en_spring_boot.md)

### H2 Database

- [Que es H2](03-Java%20Persistencia/07-h2_database/01-que_es_h2.md)
- [Configuracion de H2](03-Java%20Persistencia/07-h2_database/02-configuracion_de_h2.md)
- [H2 en testing](03-Java%20Persistencia/07-h2_database/03-h2_en_testing.md)
- [H2 Console](03-Java%20Persistencia/07-h2_database/04-h2_console.md)

### Connection Pooling

- [HikariCP](03-Java%20Persistencia/08-connection_pooling/01-hikaricp.md)
- [Configuracion por base de datos](03-Java%20Persistencia/08-connection_pooling/02-configuracion_por_bd.md)
- [Monitoreo del pool](03-Java%20Persistencia/08-connection_pooling/03-monitoreo_del_pool.md)
- [Apache DBCP2](03-Java%20Persistencia/08-connection_pooling/04-apache_dbcp2.md)
- [C3P0](03-Java%20Persistencia/08-connection_pooling/05-c3p0.md)

### Transacciones

- [Transacciones en Java](03-Java%20Persistencia/09-transacciones/01-transacciones_en_java.md)
- [Transacciones con JDBC](03-Java%20Persistencia/09-transacciones/02-transacciones_con_jdbc.md)
- [Transacciones con JPA/Hibernate](03-Java%20Persistencia/09-transacciones/03-transacciones_con_jpa_hibernate.md)
- [Propagacion de transacciones](03-Java%20Persistencia/09-transacciones/04-propagacion_de_transacciones.md)
- [Aislamiento de transacciones](03-Java%20Persistencia/09-transacciones/05-aislamiento_de_transacciones.md)
- [Transacciones distribuidas (JTA)](03-Java%20Persistencia/09-transacciones/06-transacciones_distribuidas_jta.md)

### Spring Data Jpa

- [Que es Spring Data JPA](03-Java%20Persistencia/10-spring_data_jpa/01-que_es_spring_data.md)
- [Repository interfaces](03-Java%20Persistencia/10-spring_data_jpa/02-repository_interfaces.md)
- [Query derivation](03-Java%20Persistencia/10-spring_data_jpa/03-query_derivation.md)
- [@Query annotation](03-Java%20Persistencia/10-spring_data_jpa/04-query_annotation.md)
- [Paginacion y sorting](03-Java%20Persistencia/10-spring_data_jpa/05-paginacion_y_sorting.md)
- [Custom implementations](03-Java%20Persistencia/10-spring_data_jpa/06-custom_implementations.md)
- [Projections](03-Java%20Persistencia/10-spring_data_jpa/07-projections.md)

### Testing De Persistencia

- [Testing con H2](03-Java%20Persistencia/11-testing_de_persistencia/01-testing_con_h2.md)
- [Testing con TestContainers](03-Java%20Persistencia/11-testing_de_persistencia/02-testing_con_testcontainers.md)
- [Mocking de repositorios](03-Java%20Persistencia/11-testing_de_persistencia/03-mocking_de_repositorios.md)
- [Test de transacciones](03-Java%20Persistencia/11-testing_de_persistencia/04-test_de_transacciones.md)
- [Datos de prueba](03-Java%20Persistencia/11-testing_de_persistencia/05-datos_de_prueba.md)

### Rendimiento De Persistencia

- [N+1 Problem](03-Java%20Persistencia/12-rendimiento_de_persistencia/01-n_plus_one_problem.md)
- [Batch Processing en persistencia](03-Java%20Persistencia/12-rendimiento_de_persistencia/02-batch_processing.md)
- [Paginacion en persistencia](03-Java%20Persistencia/12-rendimiento_de_persistencia/03-paginacion.md)
- [Fetch strategies](03-Java%20Persistencia/12-rendimiento_de_persistencia/04-fetch_strategies.md)
- [Connection pool tuning](03-Java%20Persistencia/12-rendimiento_de_persistencia/05-connection_pool_tuning.md)
- [SQL optimization](03-Java%20Persistencia/12-rendimiento_de_persistencia/06-sql_optimization.md)

### Manejo De Errores

- [Excepciones de persistencia](03-Java%20Persistencia/13-manejo_de_errores/01-excepciones_de_persistencia.md)
- [Manejo de errores en Spring](03-Java%20Persistencia/13-manejo_de_errores/02-manejo_de_errores_en_spring.md)
- [Logging de SQL](03-Java%20Persistencia/13-manejo_de_errores/03-logging_de_sql.md)

### Seguridad En Persistencia

- [SQL Injection](03-Java%20Persistencia/14-seguridad_en_persistencia/01-sql_injection.md)
- [Datos sensibles](03-Java%20Persistencia/14-seguridad_en_persistencia/02-datos_sensibles.md)
- [Auditoria](03-Java%20Persistencia/14-seguridad_en_persistencia/03-auditoria.md)

### Buenas Practicas

- [Diseno de entidades](03-Java%20Persistencia/15-buenas_practicas/01-diseno_de_entidades.md)
- [Repositories](03-Java%20Persistencia/15-buenas_practicas/02-repositories.md)
- [Transacciones - buenas practicas](03-Java%20Persistencia/15-buenas_practicas/03-transacciones.md)
- [Rendimiento - buenas practicas](03-Java%20Persistencia/15-buenas_practicas/04-rendimiento.md)
- [Testing - buenas practicas](03-Java%20Persistencia/15-buenas_practicas/05-testing.md)

## Java Redes

### Fundamentos De Redes

- [Paquetes y modelos de redes en Java](04-Java%20Redes/01-fundamentos_de_redes/01-paquetes_y_modelos.md)

### Sockets Tcp

- [Socket basico](04-Java%20Redes/02-sockets_tcp/01-socket_basico.md)
- [Sockets con hilos](04-Java%20Redes/02-sockets_tcp/02-sockets_con_hilos.md)
- [Configuracion avanzada de sockets](04-Java%20Redes/02-sockets_tcp/03-configuracion_avanzada.md)
- [Ejemplo completo: servidor echo](04-Java%20Redes/02-sockets_tcp/04-ejemplo_servidor_echo.md)

### Sockets Udp

- [DatagramSocket y DatagramPacket](04-Java%20Redes/03-sockets_udp/01-datagramsocket_y_datagrampacket.md)
- [Diferencias TCP vs UDP en Java](04-Java%20Redes/03-sockets_udp/02-tcp_vs_udp.md)

### Io Clasica Vs Nio

- [E/S clasica (java.io)](04-Java%20Redes/04-io_clasica_vs_nio/01-es_clasica_java_io.md)
- [NIO: Canales (Channels)](04-Java%20Redes/04-io_clasica_vs_nio/02-canales_channels.md)
- [NIO: Buffers](04-Java%20Redes/04-io_clasica_vs_nio/03-buffers.md)
- [NIO: Selectors (Multiplexores)](04-Java%20Redes/04-io_clasica_vs_nio/04-selectors.md)
- [Patron Reactor en detalle](04-Java%20Redes/04-io_clasica_vs_nio/05-patron_reactor.md)
- [Comparativa IO vs NIO](04-Java%20Redes/04-io_clasica_vs_nio/06-comparativa_io_vs_nio.md)

### Nio 2

- [File I/O con NIO.2](04-Java%20Redes/05-nio_2/01-file_io_con_nio_2.md)
- [WatchService](04-Java%20Redes/05-nio_2/02-watchservice.md)
- [AsynchronousFileChannel](04-Java%20Redes/05-nio_2/03-asynchronousfilechannel.md)

### Http Con Java

- [URL y URLConnection](04-Java%20Redes/06-http_con_java/01-url_y_urlconnection.md)
- [Peticiones HTTP](04-Java%20Redes/06-http_con_java/02-peticiones_http.md)
- [Manejo de respuestas](04-Java%20Redes/06-http_con_java/03-manejo_de_respuestas.md)

### Httpclient Moderno

- [Introduccion a HttpClient](04-Java%20Redes/07-httpclient_moderno/01-introduccion_httpclient.md)
- [Crear y configurar HttpClient](04-Java%20Redes/07-httpclient_moderno/02-crear_y_configurar.md)
- [Peticiones con HttpRequest](04-Java%20Redes/07-httpclient_moderno/03-peticiones_httprequest.md)
- [Respuestas con HttpResponse](04-Java%20Redes/07-httpclient_moderno/04-respuestas_httpresponse.md)
- [Async con HttpClient](04-Java%20Redes/07-httpclient_moderno/05-async_con_httpclient.md)

### Urls Y Uris

- [URL (Uniform Resource Locator)](04-Java%20Redes/08-urls_y_uris/01-url.md)
- [URI (Uniform Resource Identifier)](04-Java%20Redes/08-urls_y_uris/02-uri.md)
- [URLConnection vs HttpURLConnection](04-Java%20Redes/08-urls_y_uris/03-urlconnection_vs_httpurlconnection.md)

### Sockets Nio

- [SocketChannel](04-Java%20Redes/09-sockets_nio/01-socketchannel.md)
- [ServerSocketChannel](04-Java%20Redes/09-sockets_nio/02-serversocketchannel.md)
- [DatagramChannel](04-Java%20Redes/09-sockets_nio/03-datagramchannel.md)

### Programacion Asincrona

- [CompletableFuture para redes](04-Java%20Redes/10-programacion_asincrona/01-completablefuture_para_redes.md)
- [AsynchronousSocketChannel](04-Java%20Redes/10-programacion_asincrona/02-asynchronoussocketchannel.md)
- [AsynchronousServerSocketChannel](04-Java%20Redes/10-programacion_asincrona/03-asynchronousserversocketchannel.md)

### Threads Y Redes

- [Hilos en operaciones de red](04-Java%20Redes/11-threads_y_redes/01-hilos_en_operaciones_de_red.md)
- [Sincronizacion en redes](04-Java%20Redes/11-threads_y_redes/02-sincronizacion_en_redes.md)
- [Virtual Threads (Java 21+)](04-Java%20Redes/11-threads_y_redes/03-virtual_threads.md)

### Apis Rest

- [Conceptos de APIs REST](04-Java%20Redes/12-apis_rest/01-conceptos_rest.md)
- [Implementar API REST basica](04-Java%20Redes/12-apis_rest/02-implementar_api_rest.md)
- [Ejemplo completo: API CRUD](04-Java%20Redes/12-apis_rest/03-ejemplo_crud_completo.md)

### Seguridad En Redes

- [SSL/TLS en Java](04-Java%20Redes/13-seguridad_en_redes/01-ssl_tls_en_java.md)
- [HTTPS con HttpClient](04-Java%20Redes/13-seguridad_en_redes/02-https_con_httpclient.md)
- [Autenticacion en APIs](04-Java%20Redes/13-seguridad_en_redes/03-autenticacion_en_apis.md)

### Serializacion

- [JSON en Java](04-Java%20Redes/14-serializacion/01-json_en_java.md)
- [XML en Java](04-Java%20Redes/14-serializacion/02-xml_en_java.md)
- [Protocol Buffers](04-Java%20Redes/14-serializacion/03-protocol_buffers.md)

### Streaming

- [WebSockets en Java](04-Java%20Redes/15-streaming/01-websockets.md)
- [SSE (Server-Sent Events)](04-Java%20Redes/15-streaming/02-sse.md)

### Testing Y Herramientas

- [Testing de red en Java](04-Java%20Redes/16-testing_y_herramientas/01-testing_de_red.md)
- [Herramientas de diagnostico](04-Java%20Redes/16-testing_y_herramientas/02-herramientas_de_diagnostico.md)
- [Logging de red](04-Java%20Redes/16-testing_y_herramientas/03-logging_de_red.md)

### Buenas Practicas

- [Manejo de recursos](04-Java%20Redes/17-buenas_practicas/01-manejo_de_recursos.md)
- [Rendimiento](04-Java%20Redes/17-buenas_practicas/02-rendimiento.md)
- [Seguridad en redes](04-Java%20Redes/17-buenas_practicas/03-seguridad.md)
- [Manejo de errores](04-Java%20Redes/17-buenas_practicas/04-errores.md)

## Java Testing

### Fundamentos De Testing

- [Por que testing](05-Java%20Testing/01-fundamentos_de_testing/01-por_que_testing.md)
- [Tipos de testing](05-Java%20Testing/01-fundamentos_de_testing/02-tipos_de_testing.md)
- [Piramide de testing](05-Java%20Testing/01-fundamentos_de_testing/03-piramide_de_testing.md)
- [Test doubles](05-Java%20Testing/01-fundamentos_de_testing/04-test_doubles.md)
- [Red-Green-Refactor](05-Java%20Testing/01-fundamentos_de_testing/05-red_green_refactor.md)

### Junit 5

- [Que es JUnit](05-Java%20Testing/02-junit_5/01-que_es_junit.md)
- [Anotaciones basicas](05-Java%20Testing/02-junit_5/02-anotaciones_basicas.md)
- [Lifecycle callbacks](05-Java%20Testing/02-junit_5/03-lifecycle_callbacks.md)
- [Assertions](05-Java%20Testing/02-junit_5/04-assertions.md)
- [Assumptions](05-Java%20Testing/02-junit_5/05-assumptions.md)
- [Test suites](05-Java%20Testing/02-junit_5/06-test_suites.md)
- [Parametrized tests](05-Java%20Testing/02-junit_5/07-parametrized_tests.md)
- [Tests dinamicos](05-Java%20Testing/02-junit_5/08-tests_dinamicos.md)
- [JUnit 5 extensions](05-Java%20Testing/02-junit_5/09-extensions.md)

### Mockito

- [Que es Mockito](05-Java%20Testing/03-mockito/01-que_es_mockito.md)
- [Setup de Mockito](05-Java%20Testing/03-mockito/02-setup_de_mockito.md)
- [Crear mocks](05-Java%20Testing/03-mockito/03-crear_mocks.md)
- [Stubbing (when/thenReturn)](05-Java%20Testing/03-mockito/04-stubbing.md)
- [Verificacion](05-Java%20Testing/03-mockito/05-verificacion.md)
- [Argument captors](05-Java%20Testing/03-mockito/06-argument_captors.md)
- [Argument matchers](05-Java%20Testing/03-mockito/07-argument_matchers.md)
- [Spies](05-Java%20Testing/03-mockito/08-spies.md)
- [Mocks parciales](05-Java%20Testing/03-mockito/09-mocks_parciales.md)
- [Argument matchers avanzados](05-Java%20Testing/03-mockito/10-argument_matchers_avanzados.md)
- [Serializacion de mocks](05-Java%20Testing/03-mockito/11-serializacion_de_mocks.md)

### Assertj

- [Que es AssertJ](05-Java%20Testing/04-assertj/01-que_es_assertj.md)
- [Assertions basicas](05-Java%20Testing/04-assertj/02-assertions_basicas.md)
- [Strings](05-Java%20Testing/04-assertj/03-strings.md)
- [Collections](05-Java%20Testing/04-assertj/04-collections.md)
- [Exceptions](05-Java%20Testing/04-assertj/05-exceptions.md)
- [Soft assertions](05-Java%20Testing/04-assertj/06-soft_assertions.md)
- [Custom assertions](05-Java%20Testing/04-assertj/07-custom_assertions.md)

### Hamcrest

- [Que es Hamcrest](05-Java%20Testing/05-hamcrest/01-que_es_hamcrest.md)
- [Matchers basicos](05-Java%20Testing/05-hamcrest/02-matchers_basicos.md)
- [String matchers](05-Java%20Testing/05-hamcrest/03-string_matchers.md)
- [Collection matchers](05-Java%20Testing/05-hamcrest/04-collection_matchers.md)
- [Combinators](05-Java%20Testing/05-hamcrest/05-combinators.md)
- [Custom matchers](05-Java%20Testing/05-hamcrest/06-custom_matchers.md)
- [AssertJ vs Hamcrest](05-Java%20Testing/05-hamcrest/07-assertj_vs_hamcrest.md)

### Tdd

- [Que es TDD](05-Java%20Testing/06-tdd/01-que_es_tdd.md)
- [Pasos de TDD](05-Java%20Testing/06-tdd/02-pasos_de_tdd.md)
- [Reglas de TDD](05-Java%20Testing/06-tdd/03-reglas_de_tdd.md)
- [Patron Fake](05-Java%20Testing/06-tdd/04-patron_fake.md)
- [TDD avanzado](05-Java%20Testing/06-tdd/05-tdd_avanzado.md)
- [Beneficios de TDD](05-Java%20Testing/06-tdd/06-beneficios_de_tdd.md)

### Test Patterns

- [AAA (Arrange-Act-Assert)](05-Java%20Testing/07-test_patterns/01-aaa.md)
- [Given-When-Then](05-Java%20Testing/07-test_patterns/02-given_when_then.md)
- [Four-Phase Test](05-Java%20Testing/07-test_patterns/03-four_phase_test.md)
- [Builder Pattern en tests](05-Java%20Testing/07-test_patterns/04-builder_pattern.md)
- [Object Mother](05-Java%20Testing/07-test_patterns/05-object_mother.md)
- [Inline Data](05-Java%20Testing/07-test_patterns/06-inline_data.md)
- [Exhaustive Testing](05-Java%20Testing/07-test_patterns/07-exhaustive_testing.md)

### Coverage De Codigo

- [Que es coverage](05-Java%20Testing/08-coverage_de_codigo/01-que_es_coverage.md)
- [Tipos de coverage](05-Java%20Testing/08-coverage_de_codigo/02-tipos_de_coverage.md)
- [JaCoCo](05-Java%20Testing/08-coverage_de_codigo/03-jacoco.md)
- [Cobertura minima](05-Java%20Testing/08-coverage_de_codigo/04-cobertura_minima.md)
- [Mutacion Testing (PIT)](05-Java%20Testing/08-coverage_de_codigo/05-mutation_testing.md)
- [Coverage tools comparativa](05-Java%20Testing/08-coverage_de_codigo/06-coverage_tools.md)

### Property Testing

- [Que es Property-Based Testing](05-Java%20Testing/09-property_testing/01-que_es_property_based_testing.md)
- [jqwik](05-Java%20Testing/09-property_testing/02-jqwik.md)
- [Propiedades tipicas](05-Java%20Testing/09-property_testing/03-propiedades_tipicas.md)
- [Cuando usar PBT](05-Java%20Testing/09-property_testing/04-cuando_usar_pbt.md)

### Performance Testing

- [JMH (Java Microbenchmark Harness)](05-Java%20Testing/10-performance_testing/01-jmh.md)
- [Configuracion de JMH](05-Java%20Testing/10-performance_testing/02-configuracion_jmh.md)
- [Benchmarks en testing](05-Java%20Testing/10-performance_testing/03-benchmarks_en_testing.md)

### Configuracion Del Proyecto

- [Maven configuration](05-Java%20Testing/11-configuracion_del_proyecto/01-maven_configuration.md)
- [Gradle configuration](05-Java%20Testing/11-configuracion_del_proyecto/02-gradle_configuration.md)
- [Dependencias tipicas](05-Java%20Testing/11-configuracion_del_proyecto/03-dependencias_tipicas.md)
- [Estructura de test](05-Java%20Testing/11-configuracion_del_proyecto/04-estructura_de_test.md)
- [Naming conventions](05-Java%20Testing/11-configuracion_del_proyecto/05-naming_conventions.md)

### Manejo De Excepciones

- [Testing de excepciones](05-Java%20Testing/12-manejo_de_excepciones/01-testing_excepciones.md)
- [Excepciones checked](05-Java%20Testing/12-manejo_de_excepciones/02-excepciones_checked.md)
- [Excepciones custom](05-Java%20Testing/12-manejo_de_excepciones/03-excepciones_custom.md)
- [Timeout en tests](05-Java%20Testing/12-manejo_de_excepciones/04-timeout_en_tests.md)

### Testing De Codigo Legado

- [Estrategias para legacy code](05-Java%20Testing/13-testing_de_codigo_legado/01-estrategias_para_legacy.md)
- [Characterization tests](05-Java%20Testing/13-testing_de_codigo_legado/02-characterization_tests.md)
- [Michael Feathers y el codigo legado](05-Java%20Testing/13-testing_de_codigo_legado/03-feathers_legacy_code.md)
- [Refactoring con tests](05-Java%20Testing/13-testing_de_codigo_legado/04-refactoring_con_tests.md)

### Testing En La Practica

- [Test Smells](05-Java%20Testing/14-testing_en_la_practica/01-test_smells.md)
- [Buenas practicas de testing](05-Java%20Testing/14-testing_en_la_practica/02-buenas_practicas.md)
- [Test First vs Test After](05-Java%20Testing/14-testing_en_la_practica/03-test_first_vs_after.md)
- [Testing en equipo](05-Java%20Testing/14-testing_en_la_practica/04-testing_en_equipo.md)
- [Anti-patrones de testing](05-Java%20Testing/14-testing_en_la_practica/05-anti-patrones.md)
- [Maintenance de tests](05-Java%20Testing/14-testing_en_la_practica/06-maintenance.md)
