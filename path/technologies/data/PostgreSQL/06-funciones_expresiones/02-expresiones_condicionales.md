# Expresiones condicionales

El SQL relacional no seria verdaderamente util si solo pudiera seleccionar datos sin capacidad de ramificacion logica. Las expresiones condicionales permiten evaluar condiciones dentro de las consultas y devolver valores distintos segun el resultado, todo ello sin salir del motor de base de datos. Este enfoque evita transformaciones posteriores en la aplicacion y mantiene la logica de negocio cerca de los datos cuando resulta apropiado.

## CASE

`CASE` es la expresion condicional fundamental en SQL. Evalua una lista de condiciones y devuelve un resultado para la primera que se cumple. Existen dos sintaxis: `CASE simple` (compara una expresion contra valores) y `CASE searched` (evalua condiciones booleanas independientes).

### CASE searched

La forma mas flexible, donde cada `WHEN` contiene una condicion booleana completa.

```sql
SELECT nombre, salario,
    CASE
        WHEN salario < 30000 THEN 'Bajo'
        WHEN salario < 60000 THEN 'Medio'
        WHEN salario < 100000 THEN 'Alto'
        ELSE 'Muy alto'
    END AS categoria_salarial
FROM empleados;
```

`ELSE` es opcional. Si no se incluye y ninguna condicion se cumple, el resultado es `NULL`.

### CASE simple

Compara una expresion contra valores literales mediante igualdad.

```sql
SELECT nombre, departamento_id,
    CASE departamento_id
        WHEN 1 THEN 'Ventas'
        WHEN 2 THEN 'Ingenieria'
        WHEN 3 THEN 'Recursos Humanos'
        ELSE 'Otros'
    END AS nombre_departamento
FROM empleados;
```

Esta forma es mas concisa pero solo admite comparaciones de igualdad. Para rangos, subconsultas o condiciones compuestas es necesario usar la forma searched.

### CASE en clausulas UPDATE

`CASE` resulta particularmente util en operaciones de actualizacion condicional, evitando multiples sentencias `UPDATE`.

```sql
UPDATE productos
SET precio = CASE
    WHEN categoria = 'electronica' THEN precio * 0.9
    WHEN categoria = 'ropa' THEN precio * 0.7
    ELSE precio * 0.85
END;
```

### CASE anidado y expresiones booleanas

Las condiciones `WHEN` pueden contener cualquier expresion booleana valida, incluyendo `AND`, `OR`, `NOT`, subconsultas y otras funciones.

```sql
SELECT nombre, fecha_contratacion,
    CASE
        WHEN salario > 50000 AND antiguedad >= 5 THEN 'Senior Premium'
        WHEN salario > 50000 THEN 'Senior'
        WHEN antiguedad >= 5 THEN 'Experimentado'
        ELSE 'Estandar'
    END AS perfil
FROM empleados;
```

## COALESCE

`COALESCE` devuelve el primer argumento no nulo de la lista. Acepta cualquier numero de argumentos y es del tipo del argumento que finalmente se devuelve.

```sql
SELECT COALESCE(NULL, NULL, 'tercero', 'cuarto');  -- tercero
SELECT COALESCE(NULL, 0);                           -- 0
```

El caso de uso mas frecuente es sustituir valores NULL por un valor por defecto en consultas:

```sql
SELECT nombre, COALESCE(telefono, 'No registrado') AS telefono
FROM clientes;
```

Tambien es util para recorrer columnas alternativas hasta encontrar un valor presente:

```sql
SELECT producto,
    COALESCE(precio_oferta, precio_normal, precio_estimado) AS precio_final
FROM catalogo;
```

> Aunque `COALESCE` acepta multiples argumentos, solo evalua los necesarios hasta encontrar el primer no NULL. Esto tiene implicaciones de rendimiento si algun argumento es una subconsulta costosa.

La funcion `IFNULL` de otros motores SQL no existe en PostgreSQL; se reemplaza directamente por `COALESCE` con dos argumentos.

## NULLIF

`NULLIF` devuelve `NULL` si los dos argumentos son iguales; en caso contrario devuelve el primer argumento. Es el opuesto funcional de `COALESCE` en cierto sentido: donde una hace visible el primer valor no nulo, la otra oculta un valor concreto transformandolo en NULL.

```sql
SELECT NULLIF(10, 10);   -- NULL
SELECT NULLIF(10, 5);    -- 10
SELECT NULLIF('', '');   -- NULL
```

La utilidad practica aparece al evitar division por cero o al normalizar datos donde cierto valor debe tratarse como ausente:

```sql
-- Evitar division por cero
SELECT 100 / NULLIF(cantidad, 0) AS ratio
FROM ventas;

-- Tratar cadenas vacias como NULL
SELECT NULLIF(trim(telefono), '') AS telefono_limpio
FROM clientes;
```

`NULLIF` tambien se emplea en agregaciones para excluir valores especificos del conteo:

```sql
SELECT COUNT(NULLIF(estado, 'cancelado')) AS pedidos_activos
FROM pedidos;
```

## GREATEST y LEAST

`GREATEST` y `LEAST` devuelven el valor maximo y minimo de una lista de expresiones. A diferencia de las funciones de agregacion `MAX` y `MIN` que operan sobre filas, estas trabajan sobre columnas dentro de una misma fila.

```sql
SELECT GREATEST(10, 25, 8, 42);         -- 42
SELECT LEAST(10, 25, 8, 42);            -- 8

SELECT GREATEST('2026-01-01', '2026-06-15', '2026-03-20')::date;  -- 2026-06-15
```

Son especialmente practicas para calcular el valor mas reciente o mas antiguo entre varias columnas de fecha, o para establecer limites superiores e inferiores.

```sql
-- Fecha mas reciente entre varias columnas
SELECT id,
    GREATEST(fecha_creacion, fecha_actualizacion, fecha_vencimiento) AS ultima_fecha
FROM documentos;

-- Acotar un valor entre un rango
SELECT LEAST(GREATEST(puntuacion, 0), 100) AS puntuacion_acotada
FROM evaluaciones;
```

> `GREATEST` y `LEAST` devuelven NULL si cualquier argumento es NULL. Para ignorar NULLs es necesario usar `COALESCE` o filtrar previamente.

