<h1 align="center">Indices</h1>

<h2>📑 Contenido</h2>

- [Indices](#indices)
  - [Crear Indice](#crear-indice)
  - [Eliminar Indice](#eliminar-indice)
  - [Listar Indices](#listar-indices)
  - [Modificar Indice](#modificar-indice)
- [Índice Agrupado (Clustered Index)](#índice-agrupado-clustered-index)
- [Índice No Agrupado (Non-Clustered Index)](#índice-no-agrupado-non-clustered-index)
- [Índices en Múltiples Columnas](#índices-en-múltiples-columnas)
- [Índices Únicos](#índices-únicos)
- [Índices de Texto Completo (Full-Text Indexes)](#índices-de-texto-completo-full-text-indexes)

## Indices

- Los índices son estructuras que aceleran la recuperación de filas de una tabla o vista.

- Ayudan a mejorar el rendimiento de las consultas al permitir que la base de datos encuentre rápidamente las filas basadas en los valores de clave.

- Los índices pueden ser agrupados o no agrupados.

### Crear Indice

```sql
CREATE INDEX index_name
ON table_name(column_name);
```

### Eliminar Indice

```sql
DROP INDEX index_name;
```

### Listar Indices

```sql
SHOW INDEXES IN table_name;
```

### Modificar Indice

```sql
REINDEX INDEX index_name;
```

## Índice Agrupado (Clustered Index)

- El índice agrupado ordena y almacena las filas de datos en la tabla según los valores de la clave definida.

- Solo puede haber un índice agrupado por tabla, ya que las filas de datos solo pueden almacenarse en un orden.

```sql
CREATE CLUSTERED INDEX CI_EmployeeID ON Employees(EmployeeID);
```

## Índice No Agrupado (Non-Clustered Index)

- El índice no agrupado tiene una estructura separada de las filas de datos.

- Contiene los valores de la clave del índice no agrupado y cada entrada de valor de clave tiene un puntero a la fila de datos correspondiente.

```sql
CREATE NONCLUSTERED INDEX NCI_LastName ON Employees(LastName);
```

## Índices en Múltiples Columnas

Los índices en múltiples columnas se crean en más de una columna.
Pueden ser útiles para consultas que involucran combinaciones específicas de columnas.

```sql
CREATE NONCLUSTERED INDEX NCI_LastName_FirstName ON Employees(LastName, FirstName);
```

## Índices Únicos

Los índices únicos garantizan que no haya valores duplicados en la columna del índice.
Se pueden aplicar tanto a índices agrupados como no agrupados.

```sql
CREATE UNIQUE INDEX UI_Email ON Employees(Email);
```

## Índices de Texto Completo (Full-Text Indexes)

Los índices de texto completo se utilizan para búsquedas de texto en columnas grandes (como campos de texto o notas).
Permiten búsquedas eficientes basadas en palabras clave y frases.

```sql
CREATE FULLTEXT INDEX FTI_Notes ON Products(Notes);
```
