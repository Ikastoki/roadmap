# Aislamiento de transacciones

El aislamiento define como los cambios realizados por una transaccion son visibles para otras transacciones concurrentes. Cuanto mayor es el aislamiento, menor es la concurrencia y viceversa. La base de datos implementa el aislamiento mediante mecanismos de bloqueo o control de concurrencia multiversion (MVCC).

## Niveles de aislamiento

JDBC y SQL estandar definen cuatro niveles de aislamiento, que se configuran mediante el enumerado `Isolation` de Spring o directamente sobre la conexion JDBC.

### READ_UNCOMMITTED

Nivel minimo de aislamiento. Una transaccion puede leer datos modificados por otra transaccion que aun no ha sido confirmada.

```java
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public List<Pedido> pedidosDelDia() {
    return pedidoRepository.findByFecha(LocalDate.now());
}
```

Problemas que permite:
- **Dirty read**: lectura de datos no confirmados que pueden desaparecer si la otra transaccion hace rollback.

```text
Transaccion A: UPDATE saldo SET total = 100 WHERE id = 1
Transaccion B: SELECT total FROM cuentas WHERE id = 1  --> 100 (sin confirmar)
Transaccion A: ROLLBACK
Transaccion B: El valor 100 era invalido (dirty read)
```

READ_UNCOMMITTED rara vez se usa. Solo tiene sentido en escenarios donde la precision absoluta no es necesaria, como estimaciones o dashboards aproximados.

### READ_COMMITTED

Nivel por defecto en la mayoria de bases de datos (PostgreSQL, Oracle, SQL Server). Garantiza que solo se leen datos confirmados.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public Informe generarInformeDiario() {
    return new Informe(
        pedidoRepository.count(),
        facturaRepository.totalFacturado()
    );
}
```

Problemas que permite:
- **Non-repeatable read**: dentro de la misma transaccion, una misma consulta puede devolver resultados distintos si otra transaccion modifica y confirma los datos entre ambas lecturas.

```text
Transaccion A: SELECT saldo FROM cuentas WHERE id = 1  --> 100
Transaccion B: UPDATE cuentas SET saldo = 200 WHERE id = 1; COMMIT
Transaccion A: SELECT saldo FROM cuentas WHERE id = 1  --> 200 (non-repeatable read)
```

READ_COMMITTED es el equilibrio mas comun entre consistencia y rendimiento. Acepta non-repeatable reads pero evita dirty reads.

### REPEATABLE_READ

Garantiza que, dentro de una misma transaccion, una consulta devuelve los mismos resultados independientemente de las modificaciones de otras transacciones.

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void procesarLote() {
    BigDecimal total = pedidoRepository.sumarTotal();
    // Ninguna otra transaccion puede modificar los pedidos
    // hasta que esta transaccion termine
    aplicarDescuento(total);
}
```

Problemas que permite:
- **Phantom read**: una consulta que filtra por rango puede obtener filas nuevas (phantoms) si otra transaccion inserta registros que coinciden con el filtro.

```text
Transaccion A: SELECT count(*) FROM pedidos WHERE total > 100  --> 5
Transaccion B: INSERT INTO pedidos (total) VALUES (200); COMMIT
Transaccion A: SELECT count(*) FROM pedidos WHERE total > 100  --> 6 (phantom read)
```

PostgreSQL trata REPEATABLE_READ y SERIALIZABLE de forma especial: no hay phantom reads en REPEATABLE_READ porque usa MVCC con snapshots. En MySQL con InnoDB, REPEATABLE_READ tambien elimina phantom reads mediante next-key locking.

### SERIALIZABLE

Nivel maximo de aislamiento. Las transacciones se ejecutan como si fueran secuenciales, una detras de otra.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void reservarAsiento(Asiento asiento) {
    // Garantiza que no hay condiciones de carrera
    if (asientoRepository.countByOcupadoFalse() == 0) {
        throw new SinAsientosException();
    }
    asiento.setOcupado(true);
    asientoRepository.save(asiento);
}
```

Problemas:
- Alto coste de rendimiento por la serializacion forzada.
- Alta probabilidad de errores de serializacion (SQLSTATE 40001).
- Menor escalabilidad.

```java
// Reintentar transaccion SERIALIZABLE ante fallo de serializacion
@Transactional(isolation = Isolation.SERIALIZABLE)
public void reservar(Asiento asiento) {
    // El framework debe reintentar si se lanza
    // CannotSerializeTransactionException
}
```

> El nivel SERIALIZABLE no es necesario en la mayoria de aplicaciones. READ_COMMITTED combinado con bloqueos optimistas o pesimistas en las entidades problematicas suele ser suficiente y con mejor rendimiento.

## Problemas de concurrencia

### Dirty read

Lectura de datos no confirmados por otra transaccion. Solo ocurre en READ_UNCOMMITTED.

### Non-repeatable read

Lectura de un mismo registro dos veces dentro de la misma transaccion con resultados diferentes porque otra transaccion modifico el registro entre medias.

### Phantom read

Lectura de un conjunto de filas dos veces dentro de la misma transaccion con resultados diferentes porque otra transaccion inserto o elimino filas entre medias.

## Tabla resumen

| Nivel | Dirty read | Non-repeatable read | Phantom read |
|-------|-----------|-------------------|-------------|
| READ_UNCOMMITTED | Si | Si | Si |
| READ_COMMITTED | No | Si | Si |
| REPEATABLE_READ | No | No | Si |
| SERIALIZABLE | No | No | No |

## Como elegir el nivel de aislamiento

La eleccion depende del equilibrio entre consistencia y concurrencia:

- **READ_COMMITTED**: opcion por defecto para la mayoria de aplicaciones. Evita dirty reads con minimo coste de rendimiento.
- **REPEATABLE_READ**: necesario cuando una transaccion necesita leer un registro varias veces y esperar que no cambie. Tipico en procesos batch o informes.
- **SERIALIZABLE**: solo para operaciones con condiciones de carrera que no pueden resolverse con bloqueos optimistas.
- **READ_UNCOMMITTED**: desaconsejado en casi todos los escenarios.

> El nivel de aislamiento se configura en la conexion JDBC o mediante anotaciones, pero la base de datos subyacente tiene la ultima palabra. Si el motor no soporta un nivel, puede degradar al nivel mas cercano o lanzar una excepcion. PostgreSQL, por ejemplo, no tiene READ_UNCOMMITTED y lo trata como READ_COMMITTED.
