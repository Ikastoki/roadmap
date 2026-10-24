# Particionamiento en PostgreSQL

El particionamiento divide una tabla logica en segmentos fisicos (particiones) segun un criterio definido. Cada particion es una tabla independiente que hereda la estructura de la tabla padre, pero PostgreSQL las trata como un unico objeto a efectos de consulta. El objetivo principal es mejorar el rendimiento de las consultas que filtran por la clave de particion (podando particiones que no contienen datos relevantes) y facilitar la gestion de datos historicos.

## Tipos de particionamiento nativo

PostgreSQL soporta tres metodos de particionamiento declarativo desde la version 10, mejorados sustancialmente en versiones posteriores.

### PARTITION BY RANGE

Divide los datos en intervalos contiguos definidos por una o varias columnas. Es el metodo mas usado para datos temporales y secuenciales.

```sql
CREATE TABLE pedidos (
    id SERIAL,
    fecha DATE NOT NULL,
    cliente_id INTEGER,
    importe NUMERIC
) PARTITION BY RANGE (fecha);

CREATE TABLE pedidos_2024 PARTITION OF pedidos
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE pedidos_2025 PARTITION OF pedidos
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

Cada particion cubre un rango abierto en el limite inferior y cerrado en el superior (`[desde, hasta)`).

### PARTITION BY LIST

Divide los datos segun una lista de valores discretos. Es adecuado para categorias como region geografica, estado o tipo.

```sql
CREATE TABLE clientes (
    id SERIAL,
    nombre TEXT,
    region TEXT NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE clientes_norte PARTITION OF clientes
    FOR VALUES IN ('Norte', 'Noroccidente');

CREATE TABLE clientes_sur PARTITION OF clientes
    FOR VALUES IN ('Sur', 'Suroccidente');
```

Los valores no contemplados en ninguna particion provocan un error al insertar, a menos que se cree una particion por defecto:

```sql
CREATE TABLE clientes_default PARTITION OF clientes DEFAULT;
```

### PARTITION BY HASH

Distribuye los datos en un numero fijo de particiones aplicando una funcion hash a la clave. Garantiza una distribucion uniforme pero no permite poda de particiones en consultas (salvo que se filtre por el valor exacto de la clave). Es util para balancear carga de escritura.

```sql
CREATE TABLE transacciones (
    id SERIAL,
    cliente_id INTEGER NOT NULL,
    monto NUMERIC
) PARTITION BY HASH (cliente_id);

CREATE TABLE transacciones_p0 PARTITION OF transacciones
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE transacciones_p1 PARTITION OF transacciones
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE transacciones_p2 PARTITION OF transacciones
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE transacciones_p3 PARTITION OF transacciones
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## Particionamiento por herencia (legacy)

Antes del particionamiento declarativo (versiones 8.x-9.x), se usaba herencia de tablas con `CHECK` constraints y reglas o triggers para redirigir las inserciones. Este enfoque sigue funcionando pero no se recomienda para proyectos nuevos:

```sql
CREATE TABLE pedidos_2024 () INHERITS (pedidos);
ALTER TABLE pedidos_2024 ADD CHECK (fecha >= '2024-01-01' AND fecha < '2025-01-01');
```

El particionamiento declarativo es preferible porque maneja automaticamente la redireccion de inserciones, la poda de particiones en el planificador y la validacion de limites.

## ATTACH y DETACH PARTITION

`ATTACH PARTITION` incorpora una tabla existente como particion de una tabla particionada. Es util para cargar datos historicamente y luego convertirlos en particion sin interrumpir el servicio:

```sql
CREATE TABLE pedidos_2023 (LIKE pedidos INCLUDING DEFAULTS);
-- Cargar datos en pedidos_2023...
ALTER TABLE pedidos ATTACH PARTITION pedidos_2023
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
```

`DETACH PARTITION` separa una particion de la tabla padre, convirtiendola en una tabla independiente. No elimina los datos:

```sql
ALTER TABLE pedidos DETACH PARTITION pedidos_2023;
```

La operacion `DETACH CONCURRENTLY` (disponible desde PostgreSQL 15) permite separar la particion sin bloquear consultas concurrentes sobre el resto de la tabla.

## Poda de particiones (partition pruning)

El optimizador examina las condiciones `WHERE` de la consulta y determina que particiones deben ser escaneadas. Las particiones que no cumplen las condiciones se saltan completamente, lo que reduce drasticamente la E/S en tablas grandes.

```sql
EXPLAIN SELECT * FROM pedidos WHERE fecha = '2025-06-15';
```

En el plan de ejecucion solo aparecera la particion `pedidos_2025` si la fecha cae en su rango. La poda funciona con RANGE, LIST y tambien con HASH cuando se filtra por el valor exacto de la clave.

> La poda de particiones solo es efectiva cuando las condiciones de filtro son inmutables en tiempo de planificacion. Expresiones como `fecha + interval '1 day'` o funciones no inmutables inhabilitan la poda.

## Consideraciones practicas

- Los indices se crean por separado en cada particion. Desde PostgreSQL 11, se pueden definir indices en la tabla padre y se propagan automaticamente a las hijas.
- Las `PRIMARY KEY` deben incluir la clave de particion si se definen a nivel de la tabla padre.
- El numero excesivo de particiones (miles) degrada el rendimiento del planificador. El limite practico esta entre 100 y 1000 particiones, dependiendo de la complejidad de las consultas.
- Para datos temporales, una particion por mes o por trimestre suele ser el equilibrio adecuado entre granularidad de poda y numero de particiones gestionables.

