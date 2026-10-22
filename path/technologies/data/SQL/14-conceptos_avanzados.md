<h1 align="center">Conceptos Avanzados</h1>

<h2>📑 Contenido</h2>

- [Consultas recursivas](#consultas-recursivas)
- [Operaciones Pivot y Unpivot](#operaciones-pivot-y-unpivot)
  - [Pivot](#pivot)
  - [Unpivot](#unpivot)
- [SQL Dinámico](#sql-dinámico)
  - [Ejemplo MySQL](#ejemplo-mysql)
  - [SQL Server](#sql-server)

## Consultas recursivas

Las consultas recursivas, también conocidas como consultas recursivas comunes (common table expressions o CTEs en inglés), son un tipo especial de consulta en SQL que permite realizar operaciones recursivas en datos jerárquicos almacenados en una tabla. Este tipo de consulta es particularmente útil para trabajar con estructuras de datos como árboles, grafos y listas enlazadas.

La consulta recursiva se define mediante una expresión de tabla común (CTE) que se referencia a sí misma en su cláusula SELECT. La recursividad se controla mediante un caso base que termina la recursion y una relación recursiva que define cómo la consulta se auto-referencia y avanza a través de los datos.

Por ejemplo, si tienes una tabla que almacena datos jerárquicos como una estructura de árbol (por ejemplo, una tabla de empleados que incluye el ID del empleado y el ID del supervisor), puedes usar una consulta recursiva para recorrer el árbol y obtener información sobre la jerarquía.

```sql
WITH RECURSIVE JerarquiaEmpleado AS (
    -- Caso base: Selecciona los empleados raíz (sin supervisor)
    SELECT ID, Nombre, SupervisorID, 1 as Nivel
    FROM Empleados
    WHERE SupervisorID IS NULL

    UNION ALL

    -- Relación recursiva: Selecciona los empleados directamente supervisados por cada empleado encontrado en la etapa anterior
    SELECT E.ID, E.Nombre, E.SupervisorID, JE.Nivel + 1
    FROM Empleados E
    INNER JOIN JerarquiaEmpleado JE ON E.SupervisorID = JE.ID
)

-- Consulta final: Selecciona todos los empleados y su nivel en la jerarquía
SELECT * FROM JerarquiaEmpleado;
```

- La CTE "JerarquiaEmpleado" selecciona los empleados raíz (aquellos sin supervisor) como el caso base.
- Luego, la consulta se une recursivamente a sí misma para seleccionar los empleados directamente supervisados por cada empleado encontrado en la etapa anterior.
- Finalmente, la consulta final selecciona todos los empleados y su nivel en la jerarquía.

## Operaciones Pivot y Unpivot

Las operaciones Pivot y Unpivot son técnicas utilizadas en SQL para transformar datos de filas a columnas (Pivot) o de columnas a filas (Unpivot). Estas operaciones son útiles cuando necesitas reorganizar la estructura de tus datos para facilitar su análisis o presentación.

### Pivot

- La operación Pivot toma los valores únicos de una columna y los convierte en nuevas columnas en una tabla resultante.
- Es útil cuando tienes datos en una columna que deseas "pivotar" para que esos valores se conviertan en encabezados de columna.
- Se utiliza típicamente en conjuntos de datos que tienen una estructura de tabla cruzada (cross-tab).
- Puede ser utilizado para resumir y presentar datos de manera más compacta y legible.

```sql
SELECT *
FROM (
    SELECT Producto, Año, Ventas
    FROM Ventas
) AS SourceTable
PIVOT (
    SUM(Ventas)
    FOR Año IN ([2019], [2020], [2021])
) AS PivotTable;
```

### Unpivot

- La operación Unpivot hace lo contrario al Pivot: convierte columnas en filas.
- Toma los valores de múltiples columnas y los convierte en filas en una tabla resultante, con una nueva columna que indica el origen de los valores.
- Es útil cuando tienes datos en múltiples columnas que deseas "desglosar" para analizarlos más fácilmente o para normalizar la estructura de los datos.
- Puede ser útil para realizar agregaciones o análisis más detallados de datos.

```sql
SELECT Producto, Año, Ventas
FROM (
    SELECT Producto, [2019], [2020], [2021]
    FROM Ventas
) AS SourceTable
UNPIVOT (
    Ventas FOR Año IN ([2019], [2020], [2021])
) AS UnpivotTable;
```

## SQL Dinámico

El SQL dinámico es una técnica en la que las consultas SQL se generan y ejecutan en tiempo de ejecución, en lugar de estar escritas estáticamente en el código. Esto permite construir consultas SQL de manera dinámica en función de ciertas condiciones o parámetros que pueden variar.

La construcción de consultas dinámicas puede ser útil en situaciones donde las consultas estáticas no son suficientes o prácticas.

- **Generación de consultas condicionales:** Se puede construir una consulta SQL con diferentes condiciones WHERE en función de los valores de entrada o parámetros proporcionados por el usuario.

- **Construcción de consultas dinámicas complejas:** En casos donde la lógica de la consulta es compleja o requiere construcciones dinámicas de JOINs, condiciones, o incluso la selección de columnas, el SQL dinámico puede ser una solución más flexible.

- **Consulta de tablas dinámicas:** Si el nombre de la tabla o las columnas cambian dinámicamente, el SQL dinámico puede ser útil para construir consultas que se adaptan a estos cambios.

### Ejemplo MySQL

```sql
-- Declarar variables para los criterios de selección
SET @category := 'Electronics';
SET @max_price := 1000;

-- Construir la consulta dinámica
SET @query := CONCAT('SELECT * FROM productos WHERE 1=1');

IF @category IS NOT NULL THEN
    SET @query := CONCAT(@query, ' AND categoria = ''', @category, '''');
END IF;

IF @max_price IS NOT NULL THEN
    SET @query := CONCAT(@query, ' AND precio <= ', @max_price);
END IF;

-- Ejecutar la consulta dinámica
PREPARE dynamic_statement FROM @query;
EXECUTE dynamic_statement;
DEALLOCATE PREPARE dynamic_statement;
```

### SQL Server

```sql
DECLARE @Query NVARCHAR(MAX);
DECLARE @Category NVARCHAR(50);
SET @Category = 'Electronics';

SET @Query = 'SELECT ProductName, Price FROM Products WHERE Category = ''' + @Category + '''';

EXEC sp_executesql @Query;
```
