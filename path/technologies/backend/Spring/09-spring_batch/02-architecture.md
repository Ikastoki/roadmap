# Architecture

La arquitectura de Spring Batch se estructura en torno a un conjunto de componentes que gestionan desde la definicion de los trabajos hasta su ejecucion, persistencia y monitoreo. Cada componente tiene una responsabilidad clara y bien delimitada, lo que permite construir sistemas batch modulares y mantenibles.

## Flujo general de ejecucion

```
+-------------------+       +----------------+       +------------------+
|   JobLauncher     |------>|      Job       |------>|   JobRepository  |
+-------------------+       +----------------+       +------------------+
                                    |
                            +-------v--------+
                            |      Step     |
                            +----------------+
                                    |
                    +---------------+----------------+
                    |               |                 |
              +-----v-----+  +-----v------+  +------v------+
              |  Reader    |  | Processor  |  |   Writer    |
              +------------+  +------------+  +-------------+
```

## JobRepository

El JobRepository es el componente encargado de persistir toda la informacion relacionada con la ejecucion de los trabajos. Almacena registros sobre JobInstance, JobExecution, StepExecution y ExecutionContext en una base de datos relacional. Funciona como el sistema de registro central que habilita la restartabilidad, el monitoreo y la auditoria de cada ejecucion.

```java
@Configuration
@EnableBatchProcessing
public class BatchInfrastructureConfig {

    @Bean
    public JobRepository jobRepository(DataSource dataSource,
                                       PlatformTransactionManager transactionManager) {
        MapJobRepositoryFactoryBean factory = new MapJobRepositoryFactoryBean();
        factory.setDataSource(dataSource);
        factory.setTransactionManager(transactionManager);
        return factory.getObject();
    }
}
```

> En produccion se recomienda un JobRepository persistente sobre bases de datos como PostgreSQL o MySQL. La implementacion en memoria solo es apta para pruebas y desarrollo.

## JobLauncher

El JobLauncher es el componente responsable de iniciar la ejecucion de un Job con un conjunto de parametros dado. Proporciona un unico metodo `run` que recibe el Job y sus parametros de entrada. Spring Boot proporciona una implementacion por defecto que puede ejecutarse de forma programatica o a traves de triggeres programados con `@Scheduled`.

```java
@Component
public class JobTrigger {

    private final JobLauncher jobLauncher;
    private final Job importJob;

    public JobTrigger(JobLauncher jobLauncher, Job importJob) {
        this.jobLauncher = jobLauncher;
        this.importJob = importJob;
    }

    public void triggerJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();
        jobLauncher.run(importJob, params);
    }
}
```

> El parametro `timestamp` se utiliza convencionalmente como identificador unico para permitir que un mismo Job se ejecute multiples veces con parametros distintos.

## JobInstance

Un JobInstance representa una ocurrencia logica de un Job dentro de un contexto especifico. Por ejemplo, un Job de generacion de informes diarios tiene un JobInstance distinto por cada dia que se ejecuta. La identidad de un JobInstance se determina por el nombre del Job y sus parametros de identificacion.

## JobExecution

Un JobExecution representa una unica tentativa de ejecutar un JobInstance. Un mismo JobInstance puede tener multiples JobExecution si falla y se reintenta. Cada JobExecution registra informacion como la fecha de inicio, la fecha de fin, el estado (COMPLETED, FAILED, STOPPED) y cualquier excepcion que haya ocurrido.

```java
JobExecution execution = jobLauncher.run(importJob, params);
System.out.println("Estado: " + execution.getStatus());
System.out.println("Inicio: " + execution.getStartTime());
System.out.println("Fin: " + execution.getEndTime());
```

## StepExecution

Cada Step dentro de un Job genera su propio StepExecution, que registra los detalles de ejecucion de ese paso individual. Incluye contadores de lectura, procesamiento, escritura, saltos y reinicios. Estos metritos permiten analizar el rendimiento y el comportamiento de cada fase del proceso batch.

| Campo | Descripcion |
|-------|-------------|
| `readCount` | Numero de items leidos |
| `writeCount` | Numero de items escritos |
| `commitCount` | Numero de commmits realizados |
| `rollbackCount` | Numero de rollbacks realizados |
| `skipCount` | Items omitidos por politicas de skip |
| `startTime` | Momento de inicio de la ejecucion |
| `status` | Estado actual del step |

## ExecutionContext

El ExecutionContext es un mecanismo clave-value que permite almacenar y recuperar datos de estado entre ejecuciones de Steps o Jobs. Es el componente que habilita la comunicacion entre componentes del batch y la persistencia de informacion de contexto que sobrevive a reinicios del proceso.

```java
@Bean
public Tasklet customTasklet() {
    return (contribution, chunkContext) -> {
        ExecutionContext jobContext = chunkContext.getStepContext()
                .getStepExecution()
                .getJobExecution()
                .getExecutionContext();

        long lastProcessedId = jobContext.getLong("lastProcessedId", 0);
        // procesar desde lastProcessedId...

        jobContext.putLong("lastProcessedId", newLastId);
        return RepeatStatus.FINISHED;
    };
}
```

## JobExplorer

El JobExplorer proporciona un interfaz de solo lectura para consultar el historial de ejecuciones almacenadas en el JobRepository. Permite inspectar JobInstances, JobExecutions, StepExecutions y ExecutionContexts de forma programatica, lo que resulta util para construir dashboards de monitoreo o herramientas de diagnostico.

```java
@Component
public class BatchMonitor {

    private final JobExplorer jobExplorer;

    public BatchMonitor(JobExplorer jobExplorer) {
        this.jobExplorer = jobExplorer;
    }

    public List<JobExecution> getRecentExecutions(String jobName) {
        JobInstance instance = jobExplorer.getJobInstance(jobName, 0);
        return jobExplorer.getJobExecutions(instance);
    }
}
```

## JobOperator

El JobOperator expone operaciones administrativas sobre los trabajos batch. Permite detener, reiniciar, reiniciar con nuevos parametros y eliminar registros de ejecuciones del JobRepository. Es el componente apropiado para construir interfaces de administracion que controlen el ciclo de vida de los procesos batch.

```java
@Component
public class BatchAdminController {

    private final JobOperator jobOperator;

    public BatchAdminController(JobOperator jobOperator) {
        this.jobOperator = jobOperator;
    }

    public void stopExecution(long executionId) throws Exception {
        jobOperator.stop(executionId);
    }

    public void restartExecution(long executionId) throws Exception {
        jobOperator.restart(executionId);
    }
}
```

> El JobOperator y el JobExplorer son componentes complementarios: el primero gestiona operaciones de escritura sobre las ejecuciones, mientras que el segundo solo consulta informacion existente.

La interaccion entre estos componentes define el ciclo de vida completo de un proceso batch: el JobLauncher inicia el Job, este ejecuta sus Steps secuencialmente o en paralelo, cada Step utiliza Reader/Processor/Writer para transformar datos, y toda la informacion de estado se persiste en el JobRepository para habilitar restartabilidad y monitoreo. Esta arquitectura modular permite escalar desde procesos simples hasta flujos complejos con particionamiento y procesamiento multihilo.

