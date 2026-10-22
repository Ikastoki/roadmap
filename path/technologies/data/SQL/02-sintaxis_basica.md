<h1 align="center">Sintaxis SQL Básica</h1>

<h2>📑 Contenido</h2>

- [Palabras clave SQL](#palabras-clave-sql)
- [Tipo de datos](#tipo-de-datos)
- [Operadores](#operadores)
  - [Operadores Aritméticos](#operadores-aritméticos)
  - [Operadores de Comparación](#operadores-de-comparación)
  - [Operadores Lógicos](#operadores-lógicos)
  - [Operadores de Concatenación](#operadores-de-concatenación)
  - [Operadores de Asignación](#operadores-de-asignación)
  - [Operadores de Filtro](#operadores-de-filtro)
  - [Operadores de Nulo](#operadores-de-nulo)
  - [Operadores de Ordenamiento](#operadores-de-ordenamiento)
  - [Operadores de Conjuntos](#operadores-de-conjuntos)
- [Declaraciones](#declaraciones)
  - [Manipulación de datos](#manipulación-de-datos)
  - [Definición de Esquema](#definición-de-esquema)
  - [Control de Transacciones](#control-de-transacciones)
  - [Otros](#otros)

## Palabras clave SQL

- **SELECT:** Utilizado para recuperar datos de una base de datos.

- **FROM:** Especifica la tabla o tablas de las que se van a recuperar los datos.

- **WHERE:** Filtra los registros basados en una condición específica.

- **INSERT INTO:** Utilizado para insertar nuevos registros en una tabla.

- **UPDATE:** Actualiza los datos existentes en una tabla.

- **DELETE FROM:** Elimina registros de una tabla.

- **CREATE TABLE:** Crea una nueva tabla en la base de datos.

- **ALTER TABLE:** Modifica la estructura de una tabla existente.

- **DROP TABLE:** Elimina una tabla de la base de datos.

- **JOIN:** Combina datos de dos o más tablas basado en una condición de relación.

- **INNER JOIN:** Retorna registros que tienen valores coincidentes en ambas tablas.

- **LEFT JOIN:** Retorna todos los registros de la tabla izquierda y los registros coincidentes de la tabla derecha.

- **RIGHT JOIN:** Retorna todos los registros de la tabla derecha y los registros coincidentes de la tabla izquierda.

- **FULL OUTER JOIN:** Retorna registros cuando hay una coincidencia en cualquiera de las tablas.

- **ORDER BY:** Ordena los resultados de la consulta en base a una o más columnas.

- **GROUP BY:** Agrupa los registros basado en el valor de una columna.

- **HAVING:** Filtra los grupos resultantes de una consulta GROUP BY basado en una condición.

- **COUNT:** Retorna el número de filas en un conjunto de resultados.

- **SUM:** Retorna la suma de los valores de una columna.

- **AVG:** Retorna el promedio de los valores de una columna.

## Tipo de datos

En SQL, los tipos de datos se utilizan para definir el tipo de valor que puede contener cada columna de una tabla.

- **INTEGER / INT:** Números enteros.

- **BIGINT:** Números enteros grandes.

- **SMALLINT:** Números enteros pequeños.

- **FLOAT:** Números de punto flotante.

- **REAL:** Números de punto flotante de precisión simple.

- **DOUBLE PRECISION / DOUBLE:** Números de punto flotante de precisión doble.

- **DECIMAL / NUMERIC:** Números decimales precisos.

- **CHAR(n):** Cadena de caracteres de longitud fija.

- **VARCHAR(n):** Cadena de caracteres de longitud variable, con una longitud máxima de n caracteres.

- **TEXT:** Cadena de caracteres de longitud variable, con longitud máxima definida por la implementación.

- **DATE:** Fecha en formato 'YYYY-MM-DD'.

- **TIME:** Tiempo en formato 'HH:MM:SS'.

- **TIMESTAMP:** Marca de tiempo que incluye fecha y hora.

- **BOOLEAN:** Valores verdadero o falso (true / false).

- **BLOB:** Objeto binario grande para almacenar datos binarios como imágenes, archivos, etc.

- **CLOB:** Objeto de caracteres grande para almacenar datos de caracteres grandes.

- **ARRAY:** Un conjunto de valores de un mismo tipo.

- **JSON:** Almacenamiento de datos en formato JSON (disponible en algunos sistemas de bases de datos modernos).

- **XML:** Almacenamiento de datos en formato XML (disponible en algunos sistemas de bases de datos modernos).

## Operadores

### Operadores Aritméticos

- +: Suma
- -: Resta
- \*: Multiplicación
- /: División
- %: Módulo (devuelve el resto de la división)

### Operadores de Comparación

- =: Igual a
- <: Menor que
- \> : Mayor que
- <=: Menor o igual que
- \>=: Mayor o igual que
- <> o !=: No igual a

### Operadores Lógicos

- **AND:** Devuelve verdadero si ambas condiciones son verdaderas
- **OR:** Devuelve verdadero si al menos una de las condiciones es verdadera
- **NOT:** Niega una condición, convirtiendo verdadero en falso y viceversa

### Operadores de Concatenación

- **CONCAT():** Concatena dos o más cadenas
- **||:** Concatenación (en algunos sistemas de bases de datos)

### Operadores de Asignación

- **=:** Asignación de valor a una variable o columna

### Operadores de Filtro

- **IN:** Determina si un valor coincide con cualquiera de los valores de una lista o subconsulta
- **BETWEEN:** Determina si un valor está dentro de un rango especificado
- **LIKE:** Determina si un valor coincide con un patrón especificado (utilizado con comodines: % para cero o más caracteres, y \_ para un solo carácter)

### Operadores de Nulo

- **IS NULL:** Determina si un valor es nulo
- **IS NOT NULL:** Determina si un valor no es nulo

### Operadores de Ordenamiento

- **ORDER BY:** Ordena el resultado de una consulta según una o más columnas
- **ASC:** Orden ascendente (predeterminado)
- **DESC:** Orden descendente

### Operadores de Conjuntos

- **UNION:** Combina los resultados de dos o más consultas
- **INTERSECT:** Devuelve los registros que están en ambas consultas
- **EXCEPT:** Devuelve los registros de la primera consulta que no están presentes en la segunda consulta

## Declaraciones

### Manipulación de datos

- SELECT: Recupera datos de una o más tablas.

- INSERT INTO: Inserta nuevos registros en una tabla.

- UPDATE: Actualiza datos existentes en una tabla.

- DELETE FROM: Elimina registros de una tabla.

### Definición de Esquema

- CREATE TABLE: Crea una nueva tabla en la base de datos.

- ALTER TABLE: Modifica la estructura de una tabla existente.

- DROP TABLE: Elimina una tabla de la base de datos.

### Control de Transacciones

- BEGIN TRANSACTION: Inicia una transacción.

- COMMIT: Confirma una transacción y aplica los cambios realizados.

- ROLLBACK: Cancela una transacción y deshace los cambios realizados.

### Otros

- USE: Selecciona una base de datos específica en la que ejecutar las consultas.

- GRANT: Concede privilegios a usuarios o roles.

- REVOKE: Revoca privilegios previamente otorgados a usuarios o roles.
