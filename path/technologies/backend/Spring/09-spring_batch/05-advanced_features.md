# Advanced Features

Spring Batch ofrece un conjunto de funcionalidades avanzadas que permiten construir procesos batch robustos, tolerantes a fallos y escalables. Estas caracteristicas van desde politicas de manejo de errores hasta estrategias de procesamiento paralelo que distribuyen la carga entre multiples hilos o particiones.

## Skip Policies

Las skip policies determinan como el framework maneja los items que provocan excepciones durante el procesamiento. En lugar de abortar todo el Job ante un error puntual, se pueden configurar politicas que omitan los items defectuosos y continuen procesando el resto. Spring Batch define cuatro estrategias de skip:

- **SkipPolicy**: evalua si un error permite el skip basandose en la excepcion y el conteo actual.
- **ExceptionClassSkipStrategy**: omite items cuando la excepcion coincide con una clase especifica.
- **CompositeSkipPolicy**: combina multiples politicas con una logica AND/OR.
- **AlwaysSkipStrategy**: omite siempre, util solo para pruebas.

```java
@Bean
public Step skipEnabledStep(JobRepository jobRepository,
                            PlatformTransactionManager transactionManager) {
    return new StepBuilder("skipEnabledStep", jobRepository)
            .<Record, ValidRecord>chunk(100, transactionManager)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skipLimit(100)
            .skip(ValidationException.class)
            .skip(MalformedRecordException.class)
            .noSkip(DatabaseAccessException.class)
            .build();
}
```

> Las skip policies solo aplican a las fases de lectura y procesamiento. El writer no puede saltar items individuales porque opera sobre chunks completos dentro de una transaccion. Si el writer falla, se produce un rollback de todo el chunk.

## Retry Policies

Las retry policies permiten reintentar operaciones que fallan temporalmente, como errores de conexion o violaciones de restricciones de concurrencia. Spring Batch distingue entre reintentos en la fase de lectura y en la fase de procesamiento.

```java
@Bean
public Step retryStep(JobRepository jobRepository,
                      PlatformTransactionManager transactionManager) {
    return new StepBuilder("retryStep", jobRepository)
            .<Data, ProcessedData>chunk(50, transactionManager)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .retryLimit(3)
            .retry(DeadlockLoserDataAccessException.class)
            .retry(ConcurrencyFailureException.class)
            .noRetry(OptimisticLockingFailureException.class)
            .build();
}
```

La combinacion de skip y retry policies crea un sistema de manejo de errores en capas: primero se reintenta la operacion un numero determinado de veces, y si despues de los reintentos persiste el error, se evalua si el item puede ser omitido.

## Restartability

La restartabilidad es una de las caracteristicas fundamentales de Spring Batch. Cuando un Job falla, el framework almacena el estado de cada Step en el JobRepository, incluyendo el contexto de ejecucion y la posicion de cada reader y writer. Al reiniciar el Job, se restaura este estado y el procesamiento se retoma desde el ultimo checkpoint exitoso.

```java
@Bean
public Step restartableStep(JobRepository jobRepository,
                            PlatformTransactionManager transactionManager) {
    return new StepBuilder("restartableStep", jobRepository)
            .<Line, Record>chunk(200, transactionManager)
            .reader(reader())  // implementa ItemStream para guardar posicion
            .processor(processor())
            .writer(writer())  // implementa ItemStream para guardar posicion
            .allowStartIfComplete(true)
            .startLimit(3)
            .build();
}
```

El parametro `startLimit` limita el numero de veces que un Step puede iniciarse. Esto protege contra reinicios infinitos causados por errores persistentes. Si un Step excede su limite de inicio, el Job termina con estado `FAILED`.

> No todos los componentes son restartables por defecto. Los FlatFileItemReader y FlatFileItemWriter implementan ItemStream, pero los lectores personalizados deben implementar esta interfaz manualmente para guardar y restaurar su estado.

## Chunk Size Configuration

El tamano del chunk determina cuantos items se procesan antes de realizarse un commit. Este parametro tiene un impacto directo en el rendimiento y en el consumo de memoria. Chunks pequenos reducen el consumo de memoria pero incrementan la frecuencia de commits, mientras que chunks grandes mejoran el rendimiento de E/S pero pueden agotar la memoria si los items son voluminosos.

```java
@Bean
public Step optimizedStep(JobRepository jobRepository,
                          PlatformTransactionManager transactionManager) {
    return new StepBuilder("optimizedStep", jobRepository)
            .<Line, Record>chunk(500, transactionManager)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .build();
}
```

El tamano optimo del chunk depende de multiples factores: el tipo de reader (cursor vs paginado), la complejidad del processor, la latencia del writer y los recursos disponibles. Un punto de partida comun es 100-500 items por chunk, ajustando segun los resultados de pruebas de rendimiento.

## Transaction Management in Batch

La gestion de transacciones en Spring Batch es intrinseca al modelo chunk-oriented. Cada chunk completo se ejecuta dentro de una unica transaccion controlada por el `PlatformTransactionManager`. Si cualquier fase del chunk falla, se produce un rollback completo del chunk, garantizando que no queden datos a medio procesar en el destino.

```java
@Bean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}

@Bean
public Step transactionalStep(JobRepository jobRepository,
                              PlatformTransactionManager transactionManager) {
    return new StepBuilder("transactionalStep", jobRepository)
            .<Item, Output>chunk(100, transactionManager)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .transactionIsolationLevel("ISOLATION_READ_COMMITTED")
            .build();
}
```

Para los Tasklet, la transaccion se configura de forma independiente y se aplica a toda la ejecucion del tasklet. Esto permite controlar el comportamiento transaccional de operaciones que no encajan en el patron chunk.

## Multi-threaded Steps

El procesamiento multihilo permite ejecutar el reader en un hilo mientras el processor y writer operan en otro, aumentando el吞吐量 al solapar las fases de lectura y procesamiento. Spring Batch implementa esta funcionalidad a traves de `TaskExecutor` y la configuracion de `throttleLimit`.

```java
@Bean
public Step multiThreadedStep(JobRepository jobRepository,
                              PlatformTransactionManager transactionManager) {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(25);

    return new StepBuilder("multiThreadedStep", jobRepository)
            .<Item, Output>chunk(100, transactionManager)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .taskExecutor(executor)
            .throttleLimit(4)
            .build();
}
```

> El procesamiento multihilo introduce complejidad en la gestion de estado compartido. Los readers y writers que no son thread-safe deben sincronizarse manualmente o utilizarse patrones de proteccion como `SynchronizedItemStreamReader`.

## Parallel Steps

Los parallel steps ejecutan multiples Steps de forma concurrente dentro de un mismo Job. A diferencia del procesamiento multihilo que paraleliza dentro de un Step, el paralelismo a nivel de Step permite ejecutar fases completamente independientes simultaneamente.

```java
@Bean
public Job parallelJob(JobRepository jobRepository) {
    Flow flow1 = new FlowBuilder<SimpleFlow>().start(step1(jobRepository)).build();
    Flow flow2 = new FlowBuilder<SimpleFlow>().start(step2(jobRepository)).build();
    Flow flow3 = new FlowBuilder<SimpleFlow>().start(step3(jobRepository)).build();

    SimpleFlow parallelFlow = new FlowBuilder<SimpleFlow>()
            .split(new SimpleAsyncTaskExecutor())
            .add(flow1, flow2, flow3)
            .build();

    return new JobBuilder("parallelJob", jobRepository)
            .start(parallelFlow)
            .next(consolidationStep(jobRepository))
            .build();
}
```

Los parallel steps son ideales cuando un Job contiene fases independientes que pueden ejecutarse simultaneamente sin interferirse, como la extraccion de datos de multiples fuentes o la generacion de informes en paralelo.

## Remote Chunking

El remote chunking distribuye el procesamiento de un Job entre multiples maquinas. Un componente maestro coordina la lectura de datos y envia los chunks a procesadores remotos, que ejecutan el processor y writer de forma distribuida. Esta estrategia es apropiada cuando el procesamiento es computacionalmente intensivo y no puede completarse en un solo nodo en un tiempo razonable.

Spring Batch proporciona infraestructura basada en Spring Remoting y messaging (JMS, AMQP) para implementar remote chunking. El maestro utiliza un `ItemWriter` remoto que delega el procesamiento a los esclavos a traves de la red.

## Partitioning

La particionacion es la estrategia de escalado mas flexible de Spring Batch. Permite dividir un Step logico en multiples Step execution que se ejecutan en paralelo, cada uno procesando un subconjunto del dataset completo. A diferencia del remote chunking, en la particionacion cada particion es responsable de su propia lectura, procesamiento y escritura.

```java
@Bean
public Step partitionedStep(JobRepository jobRepository,
                            Partitioner filePartitioner) {
    return new StepBuilder("partitionedStep", jobRepository)
            .partitioner("workerStep", filePartitioner)
            .step(workerStep(jobRepository))
            .taskExecutor(new ThreadPoolTaskExecutor())
            .build();
}

@Bean
public Partitioner filePartitioner() {
    Partitioner partitioner = new Partitioner() {
        @Override
        public Map<String, ExecutionContext> partition(int gridSize) {
            Map<String, ExecutionContext> partitions = new HashMap<>();
            String[] files = {"data_2024_01.csv", "data_2024_02.csv", "data_2024_03.csv"};
            for (int i = 0; i < files.length; i++) {
                ExecutionContext context = new ExecutionContext();
                context.putString("file", files[i]);
                partitions.put("partition_" + i, context);
            }
            return partitions;
        }
    };
    return partitioner;
}
```

> La particionacion requiere que los componentes del batch sean stateless y thread-safe. Cada particion opera con su propio ExecutionContext aislado, por lo que no existe estado compartido entre particiones a menos que se implemente explicitamente.

La seleccion entre estas estrategias avanzadas depende del tipo de carga de trabajo. Para grandes volumenes de datos con procesamiento ligero, la particionacion es la opcion natural. Para procesamiento intensivo distribuido, remote chunking ofrece mayor flexibilidad. Los parallel steps son ideales para Jobs con fases independientes que pueden ejecutarse en simultaneo.

