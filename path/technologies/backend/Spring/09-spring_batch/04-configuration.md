# Configuration

La configuracion de Spring Batch ha evolucionado significativamente desde las definiciones XML hasta el enfoque basado en Java config que predomina en las versiones modernas. Comprender ambos paradigmas resulta util para mantener proyectos heredados y para construir nuevas aplicaciones con las mejores practicas actuales.

## Habilitacion del procesamiento batch

La anotacion `@EnableBatchProcessing` es el punto de entrada para activar la infraestructura de Spring Batch. Cuando se aplica, habilita automaticamente los beans necesarios para el funcionamiento del framework: JobRepository, JobLauncher, JobExplorer y JobOperator. Sin embargo, en versiones recientes de Spring Boot, esta anotacion ya no es estrictamente necesaria, ya que la autoconfiguracion de Spring Boot provisiona estos beans de forma predeterminada.

```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {
    // Los beans de infraestructura se proporcionan automaticamente
}
```

> Cuando se utiliza `@EnableBatchProcessing`, la autoconfiguracion de Spring Batch de Spring Boot se desactiva. Esto significa que si necesitas personalizar los beans de infraestructura, debes definirlos manualmente. En la mayoria de los casos, omitir `@EnableBatchProcessing` y confiar en la autoconfiguracion es la opcion mas sencilla.

## JobBuilderFactory y StepBuilderFactory

Spring Batch proporciona factorias que simplifican la creacion de Jobs y Steps. `JobBuilderFactory` encapsula la creacion de `JobBuilder` con el `JobRepository` ya inyectado, eliminando la necesidad de pasarlo explicitamente en cada definicion.

```java
@Configuration
public class JobDefinitionConfig {

    @Bean
    public JobBuilderFactory jobBuilderFactory(JobRepository jobRepository) {
        return new JobBuilderFactory(jobRepository);
    }

    @Bean
    public StepBuilderFactory stepBuilderFactory(JobRepository jobRepository,
                                                 PlatformTransactionManager transactionManager) {
        return new StepBuilderFactory(jobRepository, transactionManager);
    }
}
```

En las versiones mas recientes de Spring Batch (5.x), las factorias han sido reemplazadas por constructores directos que reciben el `JobRepository` como parametro, lo que hace el codigo mas explicito pero tambien mas verboso.

```java
// Enfoque moderno (Spring Batch 5.x)
@Bean
public Job importJob(JobRepository jobRepository) {
    return new JobBuilder("importJob", jobRepository)
            .start(firstStep(jobRepository))
            .next(secondStep(jobRepository))
            .build();
}
```

## Java config vs XML config

El enfoque XML fue el metodo original de configuracion de Spring Batch y aun se encuentra en proyectos legados. Los Jobs se definian en archivos XML con la namespace `batch`, donde se especificaban los componentes del Step, las transiciones y los componentes de lectura, procesamiento y escritura.

```xml
<!-- Enfoque XML (proyectos legados) -->
<batch:job id="importJob">
    <batch:step id="step1">
        <batch:tasklet>
            <batch:chunk reader="reader" processor="processor"
                         writer="writer" chunk-size="100"/>
        </batch:tasklet>
    </batch:step>
</batch:job>

<bean id="reader" class="org.springframework.batch.item.file.FlatFileItemReader">
    <property name="resource" value="classpath:input.csv"/>
    <property name="lineMapper">
        <bean class="org.springframework.batch.item.file.mapping.DefaultLineMapper">
            <property name="delimiter">
                <bean class="org.springframework.batch.item.file.transform.DelimitedLineTokenizer">
                    <property name="names" value="id,name,email"/>
                </bean>
            </property>
        </bean>
    </property>
</bean>
```

El enfoque Java config ofrece mayor flexibilidad, tipado seguro y mejor soporte para IDEs. Permite construir flujos condicionales complejos, inyectar dependencias de forma nativa y utilizar la logica de programacion estándar de Java para definir comportamientos personalizados.

```java
// Enfoque Java config moderno
@Configuration
public class FullImportJobConfig {

    @Bean
    public Job fullImportJob(JobRepository jobRepository) {
        return new JobBuilder("fullImportJob", jobRepository)
                .start(extractionStep(jobRepository))
                    .on("COMPLETED").to(transformationStep(jobRepository))
                .from(extractionStep(jobRepository))
                    .on("FAILED").fail()
                .and()
                .next(transformationStep(jobRepository))
                    .on("COMPLETED").to(loadStep(jobRepository))
                .and()
                .build();
    }

    @Bean
    public Step extractionStep(JobRepository jobRepository) {
        return new StepBuilder("extractionStep", jobRepository)
                .<RawData, CleanData>chunk(1000, transactionManager)
                .reader(extractionReader())
                .processor(extractionProcessor())
                .writer(extractionWriter())
                .build();
    }
}
```

## Definicion de Job

La definicion de un Job en Spring Batch involucra establecer su nombre, identificar si es restartable y definir el flujo de ejecucion de sus Steps. El nombre del Job es el identificador unico que se utiliza para consultas en el JobRepository y para invocaciones programaticas.

```java
@Bean
public Job resilientJob(JobRepository jobRepository) {
    return new JobBuilder("resilientJob", jobRepository)
            .preventRestart()
            .start(stepA(jobRepository))
            .on("*").to(stepB(jobRepository))
            .from(stepA(jobRepository)).on("FAILED").fail()
            .and()
            .build();
}
```

> El metodo `preventRestart()` desactiva la capacidad de reinicio de un Job. Utilizar esta opcion solo cuando la restartabilidad no sea deseable, como en procesos idempotentes que se ejecutan por calendario.

## Definicion de Step

La definicion de un Step requiere especificar el tipo de procesamiento (chunk o tasklet), los componentes involucrados y las politicas de manejo de errores. Para Steps chunk-oriented, se debe configurar el tamano del chunk, que determina cuantos items se procesan por transaccion.

```java
@Bean
public Step robustStep(JobRepository jobRepository,
                       PlatformTransactionManager transactionManager) {
    return new StepBuilder("robustStep", jobRepository)
            .<Order, OrderConfirmation>chunk(200, transactionManager)
            .reader(orderReader())
            .processor(orderProcessor())
            .writer(orderWriter())
            .faultTolerant()
            .skipLimit(50)
            .skip(ValidationException.class)
            .retryLimit(3)
            .retry(DataAccessException.class)
            .build();
}
```

## Configuracion de Spring Boot

Spring Boot simplifica la configuracion de Spring Batch a traves de propiedades en `application.yml` o `application.properties`. Estas propiedades permiten controlar aspectos como la inicializacion del esquema, la ejecucion automatica de jobs y la configuracion del datasource dedicado para el batch.

```yaml
spring:
  batch:
    job:
      enabled: true
    jdbc:
      initialize-schema: always
      table-prefix: BATCH_
      isolation-level-for-create: DEFAULT
      max-fetch-size: 100
    operator:
      enabled: true
```

La propiedad `spring.batch.job.enabled` determina si los Jobs definidos en el contexto se ejecutan automaticamente al iniciar la aplicacion. En produccion, esta propiedad se establece generalmente en `false` para controlar la ejecucion de forma programatica o a traves de programadores externos.

> Spring Batch permite configurar un datasource dedicado separado del datasource principal de la aplicacion. Esto evita que las transacciones del batch interfieran con las transacciones de la aplicacion web y permite escalar la base de datos de metadatos de forma independiente.

La configuracion adecuada de Spring Batch equilibra la simplicidad de la autoconfiguracion de Spring Boot con la flexibilidad necesaria para personalizar cada componente segun los requisitos del proceso batch. El enfoque recomendado es comenzar con la configuracion por defecto y refinarla gradualmente segun las necesidades de rendimiento y fiabilidad.

