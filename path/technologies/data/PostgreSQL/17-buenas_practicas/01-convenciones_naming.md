# Convenciones de nombrado

Las convenciones de nombrado en una base de datos no son un mero ejercicio estetico: establecen un contrato implicito que permite a cualquier desarrollador entender la estructura, las relaciones y las restricciones del esquema sin necesidad de documentacion adicional. En PostgreSQL, donde la mezcla de mayusculas y minusculas se normaliza a minusculas a menos que se usen comillas dobles, adoptar `snake_case` consistente elimina ambiguedades y errores de tipeo.

## Tablas

Los nombres de las tablas van en `snake_case` y en plural, reflejando que almacenan un conjunto de entidades. La forma plural evita confusiones con la fila individual y unifica el criterio en todo el esquema.

```sql
CREATE TABLE usuarios        -- correcto
CREATE TABLE facturas         -- correcto
CREATE TABLE items_factura    -- correcto
CREATE TABLE usuario          -- incorrecto (singular)
CREATE TABLE FacturaItems     -- incorrecto (PascalCase, sin snake_case)
```

Las tablas que modelan relaciones muchos-a-muchos suelen combinar los nombres de las tablas relacionadas en singular separados por guion bajo: `inscripciones`, `matriculas`, `usuarios_roles`. Si la tabla intermedia tiene semantica propia (como una venta que relaciona cliente y producto), se le asigna un nombre de negocio en plural (`pedidos`, `reservas`).

## Columnas

Todas las columnas se nombran en `snake_case` y en singular, salvo que representen un array o un valor multiple. El nombre debe ser descriptivo pero conciso, reflejando el dato que almacenan.

```sql
CREATE TABLE clientes (
    id              INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nombre_completo TEXT NOT NULL,
    email           TEXT UNIQUE NOT NULL,
    fecha_alta      DATE DEFAULT CURRENT_DATE,
    esta_activo     BOOLEAN DEFAULT true
);
```

### Clave primaria

La columna de clave primaria se nombra `id` cuando es autonumerica y la tabla no tiene un identificador natural dominante. En tablas debiles o cuando la clave primaria es a su vez foreign key, se usa `tabla_id` (ej: `perfiles.usuario_id`).

```sql
CREATE TABLE departamentos (
    id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY
);

CREATE TABLE empleados (
    id              INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    departamento_id INTEGER NOT NULL REFERENCES departamentos(id)
);
```

### Clave foranea

Las columnas que referencian una clave primaria de otra tabla se nombran con el nombre de la tabla referenciada en singular seguido de `_id`: `cliente_id`, `producto_id`, `departamento_id`. Esto hace explicita la relacion sin necesidad de inspeccionar la constraint.

```sql
CREATE TABLE pedidos (
    id          INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    cliente_id  INTEGER NOT NULL REFERENCES clientes(id),
    direccion_envio_id INTEGER REFERENCES direcciones(id)
);
```

## Indices

Los indices se nombran con el prefijo `idx_`, seguido del nombre de la tabla y la columna o columnas involucradas, separados por guion bajo. Para indices compuestos se listan las columnas en el orden del indice.

```sql
CREATE INDEX idx_empleados_departamento_id ON empleados (departamento_id);
CREATE INDEX idx_pedidos_cliente_id_fecha ON pedidos (cliente_id, fecha);
CREATE UNIQUE INDEX idx_usuarios_email ON usuarios (email);
```

Para indices parciales o con condiciones especificas, se anade un sufijo descriptivo al final:

```sql
CREATE INDEX idx_pedidos_estado_pendiente ON pedidos (fecha) WHERE estado = 'pendiente';
```

## Constraints

Las restricciones se nombran con el nombre de la tabla seguido de un sufijo que indica el tipo de constraint. Aunque PostgreSQL asigna nombres automaticos si no se especifican, nombrarlas explicitamente facilita la depuracion y la gestion de errores.

| Sufijo | Tipo |
|--------|------|
| `_pkey` | `PRIMARY KEY` |
| `_fkey` | `FOREIGN KEY` |
| `_unq` | `UNIQUE` |
| `_check` | `CHECK` |
| `_excl` | `EXCLUDE` |

```sql
CREATE TABLE pedidos (
    id          INTEGER,
    cliente_id  INTEGER,
    estado      TEXT,
    total       NUMERIC(10,2),
    CONSTRAINT pedidos_pkey PRIMARY KEY (id),
    CONSTRAINT pedidos_cliente_id_fkey FOREIGN KEY (cliente_id) REFERENCES clientes(id),
    CONSTRAINT pedidos_estado_check CHECK (estado IN ('pendiente', 'pagado', 'cancelado')),
    CONSTRAINT pedidos_cliente_id_estado_unq UNIQUE (cliente_id, estado)
);
```

En tablas con claves primarias compuestas o multiples foreign keys, el nombre de la tabla como prefijo permite identificar rapidamente a que constraint pertenece un error de violacion.

## Secuencias

Las secuencias generadas implicitamente por `GENERATED AS IDENTITY` heredan el nombre de la columna. Si se crean manualmente, se nombran con el sufijo `_seq`:

```sql
CREATE SEQUENCE facturas_numero_seq START 1000;
```

## Funciones y procedimientos

Las funciones de usuario se nombran en `snake_case` y en lo posible con un verbo que describa la accion (`obtener_ventas_por_periodo`, `calcular_impuestos`). No se usa prefijo `fn_` o `sp_`, ya que el lenguaje (PL/pgSQL, SQL, PL/Python, etc.) se distingue por el contexto y la firma.

```sql
CREATE FUNCTION obtener_ventas_por_periodo(desde DATE, hasta DATE)
RETURNS TABLE (producto TEXT, total NUMERIC) AS $$
    SELECT p.nombre, SUM(dv.cantidad * dv.precio)
    FROM detalle_venta dv
    JOIN productos p ON p.id = dv.producto_id
    JOIN ventas v ON v.id = dv.venta_id
    WHERE v.fecha BETWEEN desde AND hasta
    GROUP BY p.nombre;
$$ LANGUAGE sql;
```

## Buenas practicas generales

- Evitar palabras reservadas de SQL como nombre de columna o tabla (`user`, `group`, `order`, `table`). Si es inevitable, usar `snake_case` desambiguador (`usuario`, `grupo`, `orden`).
- No usar prefijos hungaros (ej: `nvarchar_nombre`) ni indicar el tipo en el nombre: el diccionario de datos ya contiene esa informacion.
- Ser consistente: una vez adoptada una convencion, aplicarla a todas las tablas del proyecto. Mezclar estilos en la misma base de datos es mas costoso que cualquier convencion particular.
- Documentar en los comentarios de tabla o columna cualquier desviacion justificada de la convencion para que no parezca un error.

