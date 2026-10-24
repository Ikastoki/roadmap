# Diseno de esquemas

La calidad de un esquema de base de datos determina en gran medida la capacidad del sistema para evolucionar, mantener la integridad de los datos y ofrecer un rendimiento predecible. Un diseno cuidadoso, fundamentado en los principios de normalizacion y en el conocimiento de las capacidades del motor, evita la mayoria de los problemas que aparecen en produccion.

## Normalizacion

La normalizacion es un proceso sistematico para organizar las tablas y sus columnas con el objetivo de reducir la redundancia y evitar anomalias en las operaciones de insercion, actualizacion y eliminacion. Se aplica en niveles progresivos, donde cada forma normal resuelve problemas especificos de la anterior.

### Primera forma normal (1NF)

Una tabla esta en 1NF cuando cada columna contiene un valor atomico (no listas ni conjuntos) y cada fila es unica (tiene clave primaria). La atomicidad debe interpretarse en el contexto del dominio: un campo `direccion` puede ser atomico aunque internamente contenga calle, numero y ciudad, si la aplicacion nunca necesita consultar esos componentes por separado.

```sql
-- Incorrecto: columna multivaluada
CREATE TABLE empleados (
    id INTEGER PRIMARY KEY,
    telefonos TEXT  -- "612345678, 698765432, 655443322"
);

-- Correcto: tabla separada
CREATE TABLE empleados_telefonos (
    empleado_id INTEGER REFERENCES empleados(id),
    telefono    TEXT NOT NULL,
    tipo        TEXT CHECK (tipo IN ('movil', 'fijo', 'fax')),
    PRIMARY KEY (empleado_id, telefono)
);
```

### Segunda forma normal (2NF)

Una tabla esta en 2NF si esta en 1NF y todos los atributos no clave dependen de la clave primaria completa, no de solo una parte. Esta condicion solo afecta a tablas con clave primaria compuesta.

```sql
-- Incorrecto: depende solo de producto_id, no de (pedido_id, producto_id)
CREATE TABLE detalle_pedido (
    pedido_id          INTEGER,
    producto_id        INTEGER,
    nombre_producto    TEXT,         -- depende solo de producto_id
    precio_unitario    NUMERIC(10,2),-- depende solo de producto_id
    cantidad           INTEGER,
    PRIMARY KEY (pedido_id, producto_id)
);

-- Correcto: se separa informacion del producto
CREATE TABLE productos (
    id               INTEGER PRIMARY KEY,
    nombre           TEXT NOT NULL,
    precio_unitario  NUMERIC(10,2) NOT NULL
);
CREATE TABLE detalle_pedido (
    pedido_id    INTEGER,
    producto_id  INTEGER REFERENCES productos(id),
    cantidad     INTEGER,
    PRIMARY KEY (pedido_id, producto_id)
);
```

### Tercera forma normal (3NF)

Una tabla esta en 3NF si esta en 2NF y ningun atributo no clave depende transitivamente de otro atributo no clave. En otras palabras, todos los atributos no clave deben depender directamente de la clave primaria y no entre si.

```sql
-- Incorrecto: ciudad depende de codigo_postal, no de la clave primaria
CREATE TABLE clientes (
    id              INTEGER PRIMARY KEY,
    nombre          TEXT NOT NULL,
    codigo_postal   TEXT,
    ciudad          TEXT,   -- depende de codigo_postal
    provincia       TEXT    -- depende de codigo_postal
);

-- Correcto: se separa informacion geografica
CREATE TABLE codigos_postales (
    codigo    TEXT PRIMARY KEY,
    ciudad    TEXT NOT NULL,
    provincia TEXT NOT NULL
);
CREATE TABLE clientes (
    id              INTEGER PRIMARY KEY,
    nombre          TEXT NOT NULL,
    codigo_postal   TEXT REFERENCES codigos_postales(codigo)
);
```

### Forma normal de Boyce-Codd (BCNF)

BCNF es una version mas estricta de la 3NF: toda dependencia funcional no trivial debe tener como determinante una superclave. Una tabla en 3NF puede no estar en BCNF cuando existen multiples claves candidatas que se solapan.

```sql
-- Tabla donde un alumno tiene un tutor, y cada tutor solo asesora en una materia.
-- Las dependencias funcionales son:
--   (alumno, materia) -> tutor (clave compuesta)
--   tutor -> materia
-- Esto viola BCNF porque tutor -> materia y tutor no es superclave.
CREATE TABLE tutorias (
    alumno   TEXT,
    materia  TEXT,
    tutor    TEXT,
    PRIMARY KEY (alumno, materia)
);

-- Solucion BCNF: separar en dos tablas
CREATE TABLE tutores (
    tutor   TEXT PRIMARY KEY,
    materia TEXT NOT NULL
);
CREATE TABLE tutorias (
    alumno  TEXT,
    tutor   TEXT REFERENCES tutores(tutor),
    PRIMARY KEY (alumno, tutor)
);
```

## Desnormalizacion

La desnormalizacion introducir redundancia de forma controlada para mejorar el rendimiento de lecturas. Se aplica solo tras haber normalizado el esquema y haber identificado, mediante metricas de rendimiento, que el costo de los joins supera el beneficio de la normalizacion.

Escenarios donde puede estar justificada:

- **Tablas de resumen o agregados**: almacenar totales precalculados (`total_factura` en la cabecera en lugar de calcularlo siempre desde el detalle).
- **Campos derivados de alta frecuencia**: columnas como `num_comentarios` en la tabla `articulos` que se actualizan mediante triggers en lugar de contar en cada consulta.
- **Particionamiento vertical**: mover columnas poco consultadas a una tabla separada para reducir el ancho de fila y mejorar el cache de buffer.

```sql
-- Desnormalizacion controlada: total_factura se actualiza via trigger
CREATE TABLE facturas (
    id            INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    cliente_id    INTEGER NOT NULL REFERENCES clientes(id),
    fecha         DATE DEFAULT CURRENT_DATE,
    total_factura NUMERIC(12,2) DEFAULT 0  -- valor desnormalizado
);

CREATE OR REPLACE FUNCTION actualizar_total_factura()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE facturas
    SET total_factura = (
        SELECT COALESCE(SUM(cantidad * precio), 0)
        FROM detalle_factura
        WHERE factura_id = NEW.factura_id
    )
    WHERE id = NEW.factura_id;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_actualizar_total
    AFTER INSERT OR UPDATE OR DELETE ON detalle_factura
    FOR EACH ROW EXECUTE FUNCTION actualizar_total_factura();
```

## Tipos de datos correctos

La eleccion del tipo de dato afecta directamente a la integridad de los datos, el espacio en disco y el rendimiento de las consultas.

| Tipo incorrecto | Problema | Tipo correcto |
|-----------------|----------|---------------|
| `VARCHAR(10)` para codigo postal | Longitud fija innecesaria, puede truncar codigos extranjeros | `TEXT` con `CHECK` opcional |
| `FLOAT` para dinero | Perdida de precision en calculos monetarios | `NUMERIC(12,2)` |
| `INTEGER` para booleanos | Permite valores 0, 1, 2... sin semantica clara | `BOOLEAN` |
| `TEXT` para fechas | No permite validacion ni operaciones de fecha | `DATE`, `TIMESTAMP`, `TIMESTAMPTZ` |
| `VARCHAR(255)` generico | Sin justificacion, heredado de MySQL | `TEXT` si no hay limite real |

```sql
CREATE TABLE transacciones (
    id          INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    importe     NUMERIC(14,4) NOT NULL CHECK (importe > 0),  -- precision bancaria
    moneda      CHAR(3) NOT NULL DEFAULT 'EUR',               -- ISO 4217
    creado_en   TIMESTAMPTZ NOT NULL DEFAULT NOW(),           -- con zona horaria
    completado_en TIMESTAMPTZ
);
```

> Usar `TIMESTAMPTZ` en lugar de `TIMESTAMP` o `DATE` para instantes temporales evita ambiguedades con cambios de zona horaria o husos diferentes. PostgreSQL almacena internamente todo en UTC y convierte al mostrar segun la configuracion de sesion.

## Restricciones de integridad

Las restricciones son el mecanismo principal para que la base de datos garantice la consistencia de los datos, independientemente del codigo de la aplicacion que los inserte.

- **`NOT NULL`** en columnas que siempre deben tener valor, en lugar de confiar en que la aplicacion las valide.
- **`CHECK`** para reglas de dominio que no pueden expresarse como tipos: `salario >= 0`, `fecha_fin > fecha_inicio`, `estado IN ('activo', 'inactivo')`.
- **`UNIQUE`** para identificaradores alternativos: `email`, `nif`, `numero_serie`.
- **`FOREIGN KEY`** para integridad referencial, siempre con el indice correspondiente en la columna que referencia.

```sql
CREATE TABLE reservas (
    id           INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    sala_id      INTEGER NOT NULL REFERENCES salas(id),
    usuario_id   INTEGER NOT NULL REFERENCES usuarios(id),
    fecha_inicio TIMESTAMPTZ NOT NULL,
    fecha_fin    TIMESTAMPTZ NOT NULL,
    CHECK (fecha_fin > fecha_inicio),
    EXCLUDE USING gist (
        sala_id WITH =,
        tstzrange(fecha_inicio, fecha_fin) WITH &&
    )
);
CREATE INDEX idx_reservas_sala_id ON reservas (sala_id);
CREATE INDEX idx_reservas_usuario_id ON reservas (usuario_id);
```

## Comentarios en tablas y columnas

Los comentarios se almacenan en el diccionario de datos y son visibles desde cualquier herramienta que se conecte a la base de datos. Documentar el esquema con `COMMENT ON` es una practica que rinde dividendos a largo plazo.

```sql
COMMENT ON TABLE clientes IS 'Personas fisicas o juridicas registradas en el sistema';
COMMENT ON COLUMN clientes.email IS 'Direccion de correo electronico principal, usada como identificador de login';
COMMENT ON COLUMN clientes.esta_activo IS 'Indica si el cliente puede realizar operaciones (baja logica)';
COMMENT ON TABLE pedidos IS 'Cabecera de pedido: datos generales de la transaccion';
COMMENT ON COLUMN pedidos.total IS 'Total desnormalizado calculado desde detalle_pedidos via trigger';
```

Para consultar los comentarios almacenados:

```sql
SELECT c.table_schema, c.table_name, c.column_name, pgd.description
FROM pg_catalog.pg_description pgd
JOIN information_schema.columns c
    ON pgd.objoid = (quote_ident(c.table_schema) || '.' || quote_ident(c.table_name))::regclass::oid
    AND pgd.objsubid = c.ordinal_position
WHERE c.table_name = 'clientes';
```

