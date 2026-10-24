# PL/pgSQL: el lenguaje procedural de PostgreSQL

El SQL es declarativo: describes *que* quieres obtener, no *como* obtenerlo. PL/pgSQL (Procedural Language/PostgreSQL Structured Query Language) extiende SQL con construcciones procedurales -- variables, estructuras de control, manejo de errores -- permitiendo escribir logica compleja dentro del propio motor de base de datos. A diferencia de las funciones SQL puras, PL/pgSQL puede evaluar expresiones, iterar sobre resultados y ejecutar bloques condicionales, todo sin salir del servidor. Esto elimina los viajes de ida y vuelta entre la aplicacion y la base de datos, reduce la latencia y permite operaciones atomicas que mantienen la integridad referencial en el mismo lugar donde residen los datos.

## Bloques anonimos con `DO`

El bloque anonimo es la unidad mas basica de PL/pgSQL: un fragmento de codigo procedural que se ejecuta y se descarta. No queda almacenado en la base de datos, lo que lo hace ideal para tareas de administracion, migraciones puntuales o pruebas rapidas.

```sql
DO $$
DECLARE
    total integer;
BEGIN
    SELECT count(*) INTO total FROM usuarios;
    RAISE NOTICE 'Usuarios registrados: %', total;
END $$;
```

La sintaxis `$$` es un *dollar quoting* que evita tener que escapar comillas simples dentro del bloque. Todo bloque se estructura con las secciones `DECLARE` (opcional, para declarar variables), `BEGIN` (donde empieza la logica) y `END` (donde termina).

## Variables, tipos y asignacion

Las variables se declaran en la seccion `DECLARE` con la sintaxis `nombre tipo [:= valor_por_defecto]`. Los tipos pueden ser cualquier tipo nativo de PostgreSQL (`integer`, `text`, `boolean`, `numeric`, `timestamp`, etc.), tipos compuestos como `RECORD`, o el tipo de una columna existente usando `%TYPE`.

```sql
DO $$
DECLARE
    contador  integer := 0;
    nombre    text;
    precio    productos.precio%TYPE;
    fila      RECORD;
BEGIN
    nombre := 'servidor primario';
    contador := contador + 1;
    SELECT * INTO fila FROM productos WHERE id = 1;
END $$;
```

La asignacion se realiza con el operador `:=`, tambien valido en SQL estandar, o con `INTO` para volcar el resultado de una consulta. No debe confundirse con el operador de comparacion `=`.

> El operador `=` en PL/pgSQL se usa para igualdad, no para asignacion. Usar `=` donde deberia ir `:=` es uno de los errores mas frecuentes al empezar con PL/pgSQL.

## Ambito y vida de las variables

Las variables declaradas en un bloque son visibles unicamente dentro de ese bloque y sus sub-bloques anidados. Un sub-bloque puede redeclarar una variable, pero la variable externa queda oculta (no modificada) durante la ejecucion del sub-bloque.

```sql
DO $$
DECLARE
    x integer := 10;
BEGIN
    DECLARE
        x integer := 20;
    BEGIN
        RAISE NOTICE 'x interno: %', x;  -- 20
    END;
    RAISE NOTICE 'x externo: %', x;      -- 10
END $$;
```

Este sombreado de variables permite aislar calculos temporales sin contaminar el contexto superior.

