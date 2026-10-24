# Test de transacciones

El manejo de transacciones es un aspecto critico en la capa de persistencia. Las pruebas de integracion deben verificar que las transacciones se comportan correctamente: que los cambios se persisten cuando se confirma la transaccion, que se revierten cuando ocurre una excepcion y que los niveles de aislamiento se respetan. Spring Boot proporciona soporte nativo para testing transaccional mediante la anotacion `@Transactional`.

## @Transactional en tests

Cuando un metodo de test esta anotado con `@Transactional`, Spring envuelve la ejecucion del test en una transaccion y, al finalizar, la revierte automaticamente. Esto garantiza que los cambios realizados durante el test no persistan y no afecten a otros tests.

```java
@DataJpaTest
class UsuarioRepositoryTest {

    @Autowired
    private UsuarioRepository usuarioRepository;

    @Test
    @Transactional
    void deberiaGuardarUsuarioEnTransaccion() {
        Usuario usuario = new Usuario("juan@email.com", "Juan", "Perez");
        usuarioRepository.save(usuario);

        Optional<Usuario> encontrado = usuarioRepository.findById(usuario.getId());
        assertThat(encontrado).isPresent();
    }
}
```

`@DataJpaTest` ya incluye `@Transactional` de forma implicita, por lo que no es necesario anadirlo explicitamente. Sin embargo, cuando se usa `@SpringBootTest` para pruebas de integracion mas amplias, la transaccionalidad debe declararse explicitamente.

## Rollback automatico

El rollback automatico es el comportamiento por defecto en los tests de Spring. Al finalizar el metodo, Spring ejecuta `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`, lo que provoca que el `EntityManager` descarte todos los cambios pendientes sin escribirlos en la base de datos.

Este comportamiento es deseable porque:
- Los tests son independientes entre si
- No se requiere limpieza manual de datos
- El orden de ejecucion de los tests no importa
- Cada test comienza con un estado conocido

```java
@Test
void elRollbackEsAutomatico() {
    usuarioRepository.save(new Usuario("test@email.com", "Test", "User"));
    // Al finalizar este metodo, el INSERT se revierte
}

@Test
void otroTestComienzaConEstadoLimpio() {
    List<Usuario> usuarios = usuarioRepository.findAll();
    assertThat(usuarios).isEmpty(); // La tabla esta vacia
}
```

## @Commit para persistir

En ocasiones es necesario conservar los cambios realizados durante un test, por ejemplo, para depurar manualmente el estado de la base de datos despues de la ejecucion. `@Commit` sobreescribe el comportamiento de rollback por defecto y confirma la transaccion:

```java
@Test
@Commit
void deberiaPersistirLosCambios() {
    Usuario usuario = new Usuario("persistente@email.com", "Persistente", "User");
    usuarioRepository.save(usuario);
    // Este cambio se conserva en la base de datos despues del test
}
```

`@Commit` puede aplicarse a nivel de metodo o de clase. Si se aplica a la clase, todos los tests de esa clase persisten sus cambios.

> `@Commit` debe usarse con precaucion. En un conjunto de tests grande, persistir cambios puede provocar dependencias entre tests y fallos intermitentes. Su uso principal es la depuracion y la generacion de datos de referencia.

## Test de aislamiento

El nivel de aislamiento de las transacciones determina como se comportan las operaciones concurrentes. Spring permite configurar el nivel de aislamiento con `@Transactional(isolation = Isolation.REPEATABLE_READ)`. En tests, se puede verificar el comportamiento bajo diferentes niveles de aislamiento.

```java
@Test
@Transactional(isolation = Isolation.READ_COMMITTED)
void deberiaLeerSoloDatosConfirmados() {
    Usuario usuario = new Usuario("test@email.com", "Test", "User");
    usuarioRepository.saveAndFlush(usuario);

    Optional<Usuario> encontrado = usuarioRepository.findById(usuario.getId());
    assertThat(encontrado).isPresent();
}
```

Para probar escenarios de concurrencia real, se pueden lanzar hilos paralelos desde el test:

```java
@Test
void deberiaManejarActualizacionConcurrente() throws Exception {
    Usuario usuario = usuarioRepository.save(new Usuario("test@email.com", "Test", "User"));

    CountDownLatch latch = new CountDownLatch(2);

    ExecutorService executor = Executors.newFixedThreadPool(2);

    Future<?> futuro1 = executor.submit(() -> {
        usuarioRepository.findById(usuario.getId()).ifPresent(u -> {
            u.setNombre("Actualizado1");
            usuarioRepository.save(u);
            latch.countDown();
        });
    });

    Future<?> futuro2 = executor.submit(() -> {
        usuarioRepository.findById(usuario.getId()).ifPresent(u -> {
            u.setNombre("Actualizado2");
            usuarioRepository.save(u);
            latch.countDown();
        });
    });

    latch.await(5, TimeUnit.SECONDS);

    usuarioRepository.flush();
    Usuario finalUsuario = usuarioRepository.findById(usuario.getId()).orElseThrow();
    assertThat(finalUsuario.getNombre()).isIn("Actualizado1", "Actualizado2");
}
```

## @TransactionConfiguration

Aunque actualmente deprecada en favor de `@Transactional` directa, `@TransactionConfiguration` permitia configurar el comportamiento transaccional por defecto a nivel de clase:

```java
// Equivalente moderno: @Transactional en cada metodo o en la clase
@Test
@Transactional(propagation = Propagation.REQUIRES_NEW)
void deberiaEjecutarEnTransaccionIndependiente() {
    // Esta transaccion es independiente de la transaccion del test
}
```

`Propagation.REQUIRES_NEW` suspende la transaccion actual y crea una nueva. Si la nueva transaccion se confirma, los cambios persisten independientemente del resultado de la transaccion padre. Esto es util para probar comportamientos de auditoria o logs que deben persistir incluso si la operacion principal falla.

## Flush manual en tests

En situaciones donde se necesita asegurar que ciertos cambios se reflejen en la base de datos dentro del mismo test (por ejemplo, para verificar una constraint o un trigger), se puede forzar el flush del `EntityManager`:

```java
@Test
void deberiaLanzarExcepcionPorEmailDuplicado() {
    usuarioRepository.saveAndFlush(new Usuario("duplicado@email.com", "Primero", "User"));

    assertThatThrownBy(() -> {
        usuarioRepository.saveAndFlush(new Usuario("duplicado@email.com", "Segundo", "User"));
    }).isInstanceOf(DataIntegrityViolationException.class);
}
```

`saveAndFlush()` fuerza la sincronizacion con la base de datos inmediatamente, permitiendo que las excepciones de integridad se capturen dentro del test.
