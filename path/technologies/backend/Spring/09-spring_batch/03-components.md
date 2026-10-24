# Components

Spring Batch define un conjunto de componentes que forman la base de cualquier proceso batch. Comprender la responsabilidad de cada uno y como interactuan entre si es esencial para construir flujos de procesamiento correctos, eficientes y mantenibles.

## Job

Un **Job** es la unidad de trabajo mas alta en la jerarquia de Spring Batch. Representa el proceso completo que se desea ejecutar, ya sea la importacion de registros desde un archivo CSV hasta la generacion de un informe financiero consolidado. Un Job esta compuesto por uno o mas Steps que se ejecutan en un orden definido.

```java
@Bean
public Job orderProcessingJob(JobRepository jobRepository,
                              Step readStep,
                              Step validateStep,
                              Step persistStep) {
    return new JobBuilder("orderProcessingJob", jobRepository)
            .start(readStep)
            .next(validateStep)
            .on("COMPLETED").to(persistStep)
            .from(validateStep).on("FAILED").to(handleErrorStep())
            .end()
            .build();
}
```

El flujo de ejecucion dentro de un Job se define mediante **flows**, que establecen las transiciones entre Steps basadas en su estado de finalizacion. Esto permite construir flujos condicionales donde el camino a seguir depende del resultado de cada Step.

## Step

Un **Step** es una fase individual dentro de un Job. Existen dos variantes principales: los Steps chunk-oriented, que procesan datos en fragmentos, y los Steps basados en Tasklet, que ejecutan una tarea unitaria. La decision entre ambos enfoques depende de si el proceso implica la transformacion masiva de registros o la ejecucion de una operacion puntual.

```java
@Bean
public Step dataValidationStep(JobRepository jobRepository,
                               PlatformTransactionManager transactionManager) {
    return new StepBuilder("dataValidationStep", jobRepository)
            .<Record, ValidatedRecord>chunk(500, transactionManager)
            .reader(recordReader(null))
            .processor(validationProcessor())
            .writer(validatedWriter())
            .build();
}
```

## ChunkOrientedTasklet

El ChunkOrientedTasklet es el nucleo del procesamiento por fragmentos. Implementa el patron reader-processor-writer donde los items se leen uno a uno hasta completar el chunk, se procesan individualmente y finalmente se escriben en bloque. Cada chunk completo se ejecuta dentro de una unica transaccion.

```java
@Bean
public Tasklet customChunkTasklet() {
    return new ChunkOrientedTasklet<String>() {
        @Override
        public RepeatStatus execute(StepContribution contribution,
                                    ChunkContext chunkContext) throws Exception {
            Chunk<? extends String> chunk = getReader().read();
            if (chunk == null || chunk.isEmpty()) {
                return RepeatStatus.FINISHED;
            }
            // procesar items del chunk
            getWriter().write(chunk);
            return RepeatStatus.CONTINUABLE;
        }
    };
}
```

## Tasklet

Un Tasklet es una tarea atómica que puede ejecutar cualquier logica que no encaje en el patron chunk-oriented. Resulta apropiado para operaciones como ejecutar comandos del sistema, limpiar directorios temporales, invocar servicios externos o ejecutar sentencias SQL directas.

```java
@Bean
public Tasklet fileCleanupTasklet() {
    return (contribution, chunkContext) -> {
        String tempDir = (String) chunkContext.getStepContext()
                .getJobParameters()
                .get("tempDir");
        Files.walk(Path.of(tempDir))
                .filter(Files::isRegularFile)
                .forEach(path -> {
                    try { Files.deleteIfExists(path); }
                    catch (IOException e) { /* log */ }
                });
        return RepeatStatus.FINISHED;
    };
}
```

## ItemReader

El ItemReader es responsable de proporcionar items de entrada al proceso chunk. Spring Batch ofrece multiples implementaciones para distintas fuentes de datos, cada una optimizada para un escenario particular.

```java
// Lectura de archivos planos con delimitador
@Bean
public FlatFileItemReader<User> flatFileReader() {
    return new FlatFileItemReaderBuilder<User>()
            .name("userReader")
            .resource(new ClassPathResource("users.csv"))
            .delimiter()
            .names("id", "name", "email", "active")
            .fieldSetMapper(fieldSet -> {
                User user = new User();
                user.setId(fieldSet.readLong("id"));
                user.setName(fieldSet.readString("name"));
                user.setEmail(fieldSet.readString("email"));
                user.setActive(fieldSet.readBoolean("active"));
                return user;
            })
            .build();
}
```

```java
// Lectura con cursor JDBC
@Bean
public JdbcCursorItemReader<Transaction> jdbcReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Transaction>()
            .name("transactionReader")
            .dataSource(dataSource)
            .sql("SELECT id, account, amount, created_at FROM transactions WHERE processed = false")
            .rowMapper((rs, rowNum) -> {
                Transaction t = new Transaction();
                t.setId(rs.getLong("id"));
                t.setAccount(rs.getString("account"));
                t.setAmount(rs.getBigDecimal("amount"));
                return t;
            })
            .build();
}
```

```java
// Lectura paginada con JPA
@Bean
public JpaPagingItemReader<Product> jpaReader EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Product>()
            .name("productReader")
            .entityManagerFactory(emf)
            .queryString("SELECT p FROM Product p WHERE p.status = 'PENDING'")
            .pageSize(100)
            .build();
}
```

> La seleccion del ItemReader adecuado impacta directamente en el rendimiento. Para grandes volumenes de datos, los readers paginados (JpaPagingItemReader) y con cursor (JdbcCursorItemReader) ofrecen mejor rendimiento que las soluciones que cargan todos los registros en memoria.

## ItemProcessor

El ItemProcessor transforma, valida o enriquece cada item leido antes de pasarlo al writer. Recibe un item de entrada y devuelve un item de salida que puede ser del mismo tipo o de uno distinto. Si el processor retorna null, el item se descarta del chunk y no se envia al writer.

```java
@Bean
public ItemProcessor<UserDTO, User> userProcessor() {
    return userDTO -> {
        if (userDTO.getEmail() == null || userDTO.getEmail().isBlank()) {
            throw new ValidationException("Email required");
        }
        User user = new User();
        user.setName(userDTO.getName().trim().toUpperCase());
        user.setEmail(userDTO.getEmail().toLowerCase());
        user.setActive(true);
        return user;
    };
}
```

## ItemWriter

El ItemWriter recibe una lista de items procesados y los persiste en el destino configurado. Al igual que los readers, ofrece multiples implementaciones adaptadas a distintos targets de escritura.

```java
// Escritura en archivo plano
@Bean
public FlatFileItemWriter<User> flatFileWriter() {
    return new FlatFileItemWriterBuilder<User>()
            .name("userWriter")
            .resource(new FileSystemResource("output/users_export.csv"))
            .delimited()
            .names("id", "name", "email", "active")
            .build();
}
```

```java
// Escritura JDBC en lote
@Bean
public JdbcBatchItemWriter<User> jdbcWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<User>()
            .dataSource(dataSource)
            .sql("INSERT INTO users (name, email, active) VALUES (:name, :email, :active)")
            .beanMapped()
            .build();
}
```

```java
// Escritura JPA
@Bean
public JpaItemWriter<User> jpaWriter(EntityManagerFactory emf) {
    JpaItemWriter<User> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}
```

## ItemStream

El ItemStream permite a los componentes del batch mantener estado entre reinicios. Los readers y writers que implementan esta interfaz pueden guardar y restaurar su posicion de procesamiento en el ExecutionContext, habilitando la restartabilidad a nivel de componente.

```java
public class RestartableCsvReader implements ItemReader<String>, ItemStream {

    private FlatFileItemReader<String> delegate;
    private long currentLine = 0;
    private long restartLine = 0;

    @Override
    public void open(ExecutionContext executionContext) {
        if (executionContext.containsKey("currentLine")) {
            restartLine = executionContext.getLong("currentLine");
        }
        delegate.open(executionContext);
        for (long i = 0; i < restartLine; i++) {
            delegate.read();
        }
    }

    @Override
    public void update(ExecutionContext executionContext) {
        executionContext.putLong("currentLine", currentLine);
    }

    @Override
    public String read() {
        currentLine++;
        return delegate.read();
    }
}
```

> El contrato de ItemStream define tres metodos: `open` para inicializar el estado, `update` para persistir el estado actual y `close` para liberar recursos. Todos los readers y writers estandar de Spring Batch implementan esta interfaz internamente.

La composicion de estos componentes permite construir desde procesos batch simples con un unico step hasta flujos complejos con multiples etapas, manejo de errores y restartabilidad completa. La clave esta en seleccionar la combinacion adecuada de reader, processor y writer para cada escenario de procesamiento.

