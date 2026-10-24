# Transacciones programaticas

Ademas del modelo declarativo basado en `@Transactional`, Spring ofrece un enfoque programatico donde el desarrollador controla explicitamente el ciclo de vida de la transaccion mediante codigo. Este enfoque se apoya en dos mecanismos principales: `TransactionTemplate` y el uso directo de `PlatformTransactionManager`. Aunque es menos comun que el modelo declarativo, tiene escenarios validos donde el control granular del comportamiento transaccional es necesario.

## TransactionTemplate

`TransactionTemplate` encapsula la logica de iniciar, commitear y hacer rollback de una transaccion, eliminando la necesidad de manejar `TransactionStatus` manualmente. Funciona como un template method: define el esqueleto de la transaccion y el desarrollador proporciona la logica de negocio como un callback.

```java
@Service
public class TransferenciaService {

    private final TransactionTemplate transactionTemplate;

    public TransferenciaService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }

    public void transferir(Long origenId, Long destinoId, BigDecimal monto) {
        transactionTemplate.execute(status -> {
            cuentaRepository.debitar(origenId, monto);
            cuentaRepository.acreditar(destinoId, monto);
            return null; // no hay valor de retorno
        });
    }
}
```

`TransactionTemplate` puede configurarse con las mismas propiedades que `@Transactional`: propagacion, aislamiento, timeout y readOnly. Esto se hace antes de invocar `execute()`.

```java
@Bean
public TransactionTemplate transferenciaTxTemplate(PlatformTransactionManager ptm) {
    TransactionTemplate template = new TransactionTemplate(ptm);
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    template.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
    template.setTimeout(10);
    template.setReadOnly(false);
    return template;
}
```

## TransactionCallback y TransactionCallbackWithoutResult

`TransactionTemplate` ofrece dos variantes del metodo `execute()`: una que recibe un `TransactionCallback<T>` y retorna un valor, y otra que recibe un `TransactionCallbackWithoutResult` y no retorna nada. La primera es util cuando la operacion transaccional produce un resultado que debe usarse fuera de la transaccion.

```java
// Con retorno de valor
public Pedido crearPedido(Pedido pedido) {
    return transactionTemplate.execute(status -> {
        Pedido guardado = pedidoRepository.save(pedido);
        inventarioService.reservar(guardado.getProductoId(), guardado.getCantidad());
        return guardado;
    });
}

// Sin retorno de valor
public void eliminarPedidosVencidos() {
    transactionTemplate.executeWithoutResult(status -> {
        List<Pedido> vencidos = pedidoRepository.findVencidos();
        pedidoRepository.deleteAll(vencidos);
    });
}
```

La ventaja de usar `TransactionCallback` es que la excepcion lanzada dentro del callback provoca rollback automaticamente, igual que con `@Transactional`. No es necesario manejar manualmente `commit()` o `rollback()`: el template se encarga de ello.

> Cuando la lambda lanza una checked exception, el template la envuelve en un `RuntimeException` (specifique `UncheckedException` en la firma de la interfaz funcional). Si necesita propagar una checked exception especifica, declare la lambda con `throws Exception` y maneje la conversion internamente.

## Uso Directo de PlatformTransactionManager

El enfoque mas explicito y manual consiste en obtener un `PlatformTransactionManager` e invocar sus metodos directamente. Este patron se usa raramente en aplicaciones nuevas porque duplica logica que `TransactionTemplate` maneja automaticamente, pero es importante entenderlo porque es la base sobre la que se construyen los demas mecanismos.

```java
@Service
public class ImportacionService {

    private final PlatformTransactionManager transactionManager;

    public ImportacionService(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public ResultadoImportacion importarDatos(InputStream input) {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        def.setName("importacionTx");
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

        TransactionStatus status = transactionManager.getTransaction(def);
        try {
            List<Registro> registros = parsear(input);
            for (Registro reg : registros) {
                repository.save(reg);
            }
            transactionManager.commit(status);
            return ResultadoImportacion.exito(registros.size());
        } catch (Exception e) {
            transactionManager.rollback(status);
            return ResultadoImportacion.fallo(e.getMessage());
        }
    }
}
```

Este enfoque ofrece control total sobre cuando commitear o hacer rollback, e incluso permite decidir dinamicamente segun la logica de negocio. Sin embargo, es propenso a errores si no se maneja correctamente el flujo de excepciones, especialmente cuando hay multiples puntos de posible fallo.

```java
public void importacionCompleta(List<Fuente> fuentes) {
    for (Fuente fuente : fuentes) {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        TransactionStatus status = transactionManager.getTransaction(def);
        try {
            procesarFuente(fuente);
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            log.error("Error procesando fuente: " + fuente.getNombre(), e);
            // Continua con la siguiente fuente
        }
    }
}
```

## TransactionTemplate vs @Transactional

`TransactionTemplate` ofrece ventajas sobre `@Transactional` en escenarios donde el comportamiento transaccional debe ser dinamico o condicional. Por ejemplo, si la decision de iniciar una transaccion depende de un valor de configuracion que solo se conoce en tiempo de ejecution, o si se necesita crear transacciones con propiedades diferentes segun la cantidad de registros a procesar.

```java
public void procesarLote(List<Item> items) {
    // Procesar en lotes pequenos con transacciones independientes
    Lists.partition(items, 100).forEach(lote -> {
        transactionTemplate.execute(status -> {
            lote.forEach(item -> repository.save(item));
            return null;
        });
    });
}
```

Con `@Transactional`, todas las llamadas al metodo se ejecutarian en una unica transaccion. Con `TransactionTemplate`, se crean multiples transacciones independientes, reduciendo el tamano del rollback y el tiempo que los locks permanecen abiertos.

> En el 95% de los casos, `@Transactional` es suficiente y mas legible. Reserve `TransactionTemplate` para cuando necesite logica transaccional dinamica, procesamiento por lotes con transacciones independientes o control granular del ciclo de vida de la transaccion.

