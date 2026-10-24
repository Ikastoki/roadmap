# Vistas materializadas

Una vista materializada es un objeto de base de datos que, a diferencia de una vista convencional, almacena fisicamente el resultado de la consulta en disco. Esto la convierte en una estructura hibrida entre una vista y una tabla: se define como una consulta (como una vista) pero ocupa espacio real y puede indexarse (como una tabla). La compensacion es inmediata: las lecturas son drasticamente mas rapidas porque no ejecutan la consulta subyacente, pero los datos quedan congelados en el momento de la ultima actualizacion y requieren una operacion explicita para refrescarse.

## Creacion y refresco

### CREATE MATERIALIZED VIEW

La sintaxis es analoga a `CREATE VIEW` pero anadiendo la palabra `MATERIALIZED`:

```sql
CREATE MATERIALIZED VIEW resumen_ventas_mensual AS
SELECT date_trunc('month', p.fecha) AS mes,
       c.categoria,
       COUNT(p.id) AS num_pedidos,
       SUM(p.total) AS ingresos
FROM pedidos p
JOIN productos pr ON pr.id = p.producto_id
JOIN categorias c ON c.id = pr.categoria_id
GROUP BY date_trunc('month', p.fecha), c.categoria
ORDER BY mes, c.categoria;
```

La primera vez que se ejecuta `CREATE MATERIALIZED VIEW`, PostgreSQL llena la vista con los datos resultantes de la consulta. A partir de ese momento, los datos permanecen estaticos hasta que se refrescan explicitamente.

### REFRESH MATERIALIZED VIEW

Para actualizar el contenido de una vista materializada con los datos mas recientes de las tablas base:

```sql
REFRESH MATERIALIZED VIEW resumen_ventas_mensual;
```

Durante la ejecucion de `REFRESH`, PostgreSQL bloquea la vista materializada para escritura (con un `ACCESS EXCLUSIVE LOCK`), lo que impide cualquier consulta concurrente hasta que termine la operacion. Si la vista contiene millones de filas, la ventana de indisponibilidad puede ser significativa.

### REFRESH CONCURRENTLY

Para evitar el bloqueo completo, PostgreSQL ofrece la opcion `CONCURRENTLY`, que permite realizar lecturas concurrentes mientras se refresca la vista. El requisito indispensable es que la vista materializada tenga al menos un indice `UNIQUE`:

```sql
CREATE UNIQUE INDEX idx_resumen_ventas_mensual ON resumen_ventas_mensual (mes, categoria);

REFRESH MATERIALIZED VIEW CONCURRENTLY resumen_ventas_mensual;
```

Con `CONCURRENTLY`, PostgreSQL crea una version temporal de los datos, la compara con la existente mediante el indice unico, aplica solo las diferencias (inserciones, actualizaciones, eliminaciones) y commuta la version nueva en una transaccion breve. El resultado es que las lecturas nunca se interrumpen, aunque el proceso consume mas recursos (CPU, E/S y espacio temporal) que un `REFRESH` simple.

> `REFRESH MATERIALIZED VIEW CONCURRENTLY` requiere que la vista materializada tenga un indice unico. Sin el, la operacion falla. Este indice debe cubrir todas las columnas que identifican de forma univoca cada fila.

## Indices en vistas materializadas

Al almacenar datos fisicamente, las vistas materializadas admiten indices exactamente igual que las tablas ordinarias. Los indices pueden acelerar las consultas que filtran, ordenan o agrupan por columnas especificas:

```sql
CREATE INDEX idx_resumen_mes ON resumen_ventas_mensual (mes);
CREATE INDEX idx_resumen_ingresos ON resumen_ventas_mensual (ingresos DESC);
```

Los indices se mantienen automaticamente durante el `REFRESH`. Cuando se usa `CONCURRENTLY`, los indices existentes sirven ademas para acelerar la deteccion de diferencias entre los datos nuevos y los antiguos.

La estrategia de indexacion sobre vistas materializadas sigue los mismos principios que sobre cualquier tabla: analizar los patrones de consulta, identificar las columnas mas filtradas y crear indices B-tree, GiN o BRIN segun corresponda.

## Casos de uso

### Reportes y dashboards

Las vistas materializadas son la herramienta ideal para paneles de control y reportes periodicos. Un dashboard que muestra metricas de ventas del mes no necesita ver los datos del ultimo segundo; con una actualizacion cada hora o cada dia es suficiente. La vista materializada precalcula las agregaciones, y las consultas del dashboard se resuelven con lecturas secuenciales o indexadas que tardan milisegundos, no segundos:

```sql
CREATE MATERIALIZED VIEW dashboard_metricas AS
SELECT
  COUNT(DISTINCT usuario_id) AS usuarios_activos,
  COUNT(*) AS sesiones_hoy,
  AVG(duracion) AS duracion_media_minutos
FROM sesiones
WHERE fecha = current_date;

REFRESH MATERIALIZED VIEW dashboard_metricas;
```

### Procesos ETL y almacenes de datos

En entornos de extraccion, transformacion y carga, las vistas materializadas sirven como capas intermedias que materializan transformaciones costosas. Un proceso ETL puede escribir en tablas staging y luego refrescar vistas materializadas que alimentan las tablas finales del data warehouse, evitando repetir calculos complejos en cada consulta.

### Datos geoespaciales y de series temporales

Las consultas sobre datos geoespaciales (calcular distancias, intersecciones) o series temporales (ventanas moviles, interpolaciones) suelen ser intensivas en computo. Una vista materializada puede precacular estos valores y exponerlos como una tabla plana que las aplicaciones consumen sin incurrir en el coste computacional cada vez:

```sql
CREATE MATERIALIZED VIEW distancias_almacenes AS
SELECT a.id AS almacen_origen,
       b.id AS almacen_destino,
       ST_Distance(a.geom, b.geom) AS distancia_km
FROM almacenes a, almacenes b
WHERE a.id < b.id;
```

> Las vistas materializadas no son adecuadas para datos que cambian con frecuencia minima y requieren consistencia en tiempo real. En esos escenarios es mejor usar vistas convencionales o replicacion logica. La decision entre vista y vista materializada debe basarse en la frecuencia de actualizacion de los datos fuente frente a la tolerancia a la obsolescencia del consumidor.

